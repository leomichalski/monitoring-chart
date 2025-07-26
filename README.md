## Setup

### Loki S3 Storage Access

#### MinIO Example

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: loki-s3-credentials
stringData:
  S3_LOKI_S3: "null"
  S3_LOKI_ENDPOINT: http://minio.example-minio
  S3_LOKI_REGION: "null"
  S3_LOKI_ACCESS_KEY_ID: minio
  S3_LOKI_SECRET_ACCESS_KEY: minio123
  S3_LOKI_S3FORCEPATHSTYLE: "true"
  S3_LOKI_INSECURE: "true"
EOF
```

#### AWS S3 Example

TODO

### Grafana Admin Credentials

#### Automatic Generation

Just install the chart, then get the "admin" user password with the command:

```bash
kubectl get secrets grafana-admin -o yaml -o jsonpath="{.data.admin-password}" | base64 -d
```

#### Manual Configuration

Disable External Secrets for this chart, by setting "externalsecrets.enabled=false" and "externalsecrets.create=false".

Then, apply the following Secret, substituting "admin" and "prom-operator" (default values for user and password).

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: grafana-admin
type: Opaque
stringData:
  admin-user: admin
  admin-password: prom-operator
EOF
```

### Restrict promtail to collect only some logs based on a label

For instance, here's the `values.yaml` file for restricting promtail to pods labelled with "app=example-django".

```yaml
promtail:
  # -- Promtail configuration to scrape logs from pods with a specific label.
  #    This example targets pods with the label `app: my-app`.
  config:
    snippets:
      scrapeConfigs: |
        - job_name: kubernetes-pods-labeled
          kubernetes_sd_configs:
            - role: pod
          # TODO: parameterize
          relabel_configs:
            - source_labels:
                - __meta_kubernetes_pod_label_app
              action: keep
              regex: example-django # Change this to the label value you want to scrape
```

### Small setup for dev

```bash
# clone this repository
git clone https://github.com/leomichalski/anotaplaca-monitoring-chart

# navigate to the repo root folder
cd anotaplaca-monitoring-chart

# build dependencies
helm dependency build

# install the chart
helm upgrade --install monitoring . --create-namespace \
  --set loki.read.autoscaling.enabled=false \
  --set loki.write.autoscaling.enabled=false \
  --set loki.backend.autoscaling.enabled=false \
  --set loki.read.replicas=1 \
  --set loki.write.replicas=1 \
  --set loki.backend.replicas=1 \
  --set loki.loki.commonConfig.replication_factor=1 \
  --set loki.loki.storage_config.aws.bucketnames=loki-chunks \
  --set loki.loki.storage.bucketNames.chunks=loki-chunks \
  --set loki.loki.storage.bucketNames.ruler=loki-ruler
```

### Telegram bot (after the setup)

#### Getting a bot token

The primary tool for creating and managing Telegram bots is the "BotFather," an official bot provided by Telegram.

Step-by-step instructions:

1. Open Telegram and find BotFather: In the Telegram app, use the search bar to find "@BotFather". Look for the official bot, which has a blue checkmark next to its name.[1]
1. Start a chat with BotFather: Tap on the BotFather to open a chat window and click the "Start" button or type /start. This will display a list of commands.
1. Create a new bot: Type /newbot and send it. BotFather will then ask you to choose a name for your bot. This is the name your subscribers will see.[1][2]
1. Set a unique username: After giving it a name, you need to choose a username for your bot. This username must be unique and end with the word "bot" (e.g., "MyTestBot").[1][2]
1. Receive your bot token: Once you've successfully chosen a unique username, BotFather will generate a unique API token for your bot. This token is a long string of characters and is essential for controlling your bot, so keep it secure and do not share it publicly.[3][4]

#### Getting a chat ID

A chat ID is a unique identifier for a specific chat on Telegram. This ID is necessary if you want your bot to send messages to a specific user, group, or channel.

##### For a personal chat ID:

1. Search for "@RawDataBot" in the Telegram search bar and start a chat with it.
1. After starting the bot, it will display a JSON message containing your chat information.
1. Look for the "chat" object in the message, and within that, you will find the "id". This number is your chat ID.

#### For a group or channel chat ID:

1. Add a bot to the group or channel: You can add the @RawDataBot to the group or channel.
1. View the bot's message: The bot will post a message containing the chat information.
1. Find the chat ID: In the message from the bot, look for the "chat" object. The "id" within this object is the group or channel's chat ID. For group chats, the ID will typically start with a hyphen. To use this ID in some APIs, you might need to prefix it with "-100".

Create the following Secret, substituting the BOT_TOKEN.

```yaml
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: telegram-bot-secret
type: Opaque
stringData:
  bot_token: BOT_TOKEN
EOF
```

Create the Alert Manager configuration, substituting the CHAT_ID with the id of the chat where the bot will send the alerts.

```yaml
kubectl apply -f - <<EOF
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: alertmanagerconfig
  labels:
    alertmanagerConfig: global
spec:
  route:
    receiver: 'telegram-receiver'
  receivers:
  - name: 'telegram-receiver'
    telegramConfigs:
    - apiURL: 'https://api.telegram.org'
      botToken:
        key: bot_token
        name: telegram-bot-secret
      chatID: CHAT_ID
EOF
```




