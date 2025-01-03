# Alertmanager Webhook Receiver to send notifications to Slack, Discord, Telegram, and PagerDuty

[![Python Version: 3.12](
https://img.shields.io/badge/Python%20application-v3.12-blue
)](https://www.python.org/downloads/release/python-3128/)


## Prerequisites

1. Install [ngrok](https://ngrok.com/).
```bash
choco install ngrok
```
2. Ensure your System Python3 version is 3.12.
```bash
python3 -V
```
3. If your System Python is not 3.12:
```bash
choco install python@3.12
choco link python@3.12
```
4. [Create a new Slack App] https://api.slack.com/apps/, provide it with oAuth and also the chat:write permission. 
   Create a channel. Configure the App to send notification to the channel

5.[Create a new Discord App](https://discord.com/developers/applications).
   Create your Discord channel where you want to receive your
   Alertmanager notifications.
   
6. Configure Alertmanager to send notifications to that channel.

7. Create a configuration file called `config.yml` in the same directory
   as the webhook script that looks like this:
```yml
---
slack:
  bot_username: YOUR_SLACK_BOT_TOKEN
  bot_token: YOUR_SLACK_TOKEN
  environments:
    prod:
      warning:
        channel_id: YOUR_SLACK_ALERT_CHANNEL_ID
        author:
          name: Alertmanager
          icon_url: https://www.clipartmax.com/png/small/118-1186067_prometheus-software-logo-prometheus-monitoring.png
      critical:
         channel_id: YOUR_SLACK_ALERT_CHANNEL_ID
         author:
            name: Alertmanager
            icon_url: https://www.clipartmax.com/png/small/118-1186067_prometheus-software-logo-prometheus-monitoring.png
    test:
       warning:
          channel_id:YOUR_SLACK_ALERT_CHANNEL_ID
          author:
             name: Alertmanager
             icon_url: https://www.clipartmax.com/png/small/118-1186067_prometheus-software-logo-prometheus-monitoring.png
       critical:
          channel_id: YOUR_SLACK_ALERT_CHANNEL_ID
          author:
             name: Alertmanager
             icon_url: https://www.clipartmax.com/png/small/118-1186067_prometheus-software-logo-prometheus-monitoring.png

discord:
  bot_token: YOUR_DISCORD_TOKEN
  environments:
    prod:
      warning:
        channel_id: YOUR_DISCORD_CHANNEL_ID
        author:
          name: Alertmanager
          icon_url: https://www.clipartmax.com/png/small/118-1186067_prometheus-software-logo-prometheus-monitoring.png
      critical:
         channel_id: YOUR_DISCORD_CHANNEL_ID
         author:
            name: Alertmanager
            icon_url: https://www.clipartmax.com/png/small/118-1186067_prometheus-software-logo-prometheus-monitoring.png
    test:
       warning:
          channel_id: YOUR_DISCORD_CHANNEL_ID
          author:
             name: Alertmanager
             icon_url: https://www.clipartmax.com/png/small/118-1186067_prometheus-software-logo-prometheus-monitoring.png
       critical:
          channel_id: YOUR_DISCORD_CHANNEL_ID
          author:
             name: Alertmanager
             icon_url: https://www.clipartmax.com/png/small/118-1186067_prometheus-software-logo-prometheus-monitoring.png

telegram:
  bot_token: YOUR_TELEGRAM_BOT_TOKEN
  environments:
     prod:
        warning:
           chat_id: YOUR_TELEGRAM_WARNINGS_CHAT_ID
        critical:
           chat_id: YOUR_TELEGRAM_WARNINGS_CHAT_ID

pagerduty:
   environments:
      - prod
   services:
      default: f00df00df00df00df00df00df00df00d
      website: d00fd00fd00fd00fd00fd00fd00fd00f

valid_environments:
   - test
   - prod

default_environment: prod

environment_mapping:
  ap-southeast-3: prod
  ap-southeast-3: test
```

## Alertmanager Configuration
### Alertmanager
```yaml
alertmanager_route:
  repeat_interval: 8736h
  group_by: ['alertname', 'cluster', 'service']
  receiver: webhook-warning
  routes:
    - match:
        severity: webhook-critical
      receiver: webhook-critical
    - match:
        severity: webhook-warning
      receiver: webhook-warning

alertmanager_receivers:
  - name: webhook-critical
    webhook_configs:
      - url: "https://3e602e00.execute-api.ap-southeast-3.amazonaws.com/alertmanager/critical"
        send_resolved: true
  - name: webhook-warning
    webhook_configs:
       - url: "https://3e602e00.execute-api.ap-southeast-3.amazonaws.com/alertmanager/warning"
         send_resolved: true
```
### Prometheus Rules
```yaml
groups:
  - name: haproxy.rules
    rules:
      - alert: HAProxyDown
        expr: haproxy_up == 0
        for: 5m
        labels:
          severity: webhook-critical
        annotations:
          summary: "HAProxy load balancer down"
          description: "{{ $labels.job }} on {{ $labels.instance }} has been down for 5 minutes."
```

## Testing your Webhook

1. Run the webhook receiver from your terminal.
```bash
python3 webhook.py
```
2. Open a new terminal window and use [ngrok](https://ngrok.com/) to create
a URL that is publically accessible through the internet by creating a tunnel
to the webhook receiver that is running on your local machine.
```bash
ngrok http 8090
```
3. Note that the ngrok URL will change if you stop ngrok and run it again,
   so keep it running in a separate terminal window, otherwise you will not
   be able to test your webhook successfully.
4. Update your Alertmanager webhook configuration to the URL that is displayed
while ngrok is running **(be sure to use the https one)**.
5. Trigger an Alertmanager event to trigger the notification webhook (this can
   be done by running the `test.py` script provided within this project.
6. Check your Discord and Telegram channels that you created for your Alertmanager
   notifications.

## Deployment to AWS Lambda

1. Create a Python 3.12 Virtual Environment:
```bash
python3 -m venv venv/py3.12
source venv/py3.12/bin/activate
```
2. Upgrade pip.
```bash
python3 -m pip install --upgrade pip
```
3. Install the Python dependencies that are required by the Webhook receiver:
```bash
pip3 install -r requirements.txt
```
4. Create a file called `zappa_settings.json` and insert the JSON content below
to configure your AWS Lambda deployment:
```json
{
    "alertmanager": {
        "app_function": "webhook.app",
        "aws_region": "us-east-1",
        "lambda_description": "Webhook to handle Alertmanager notifications",
        "profile_name": "default",
        "project_name": "alertmanager-webhook",
        "runtime": "python3.12",
        "s3_bucket": "alertmanager-webhooks",
        "tags": {
            "service": "alertmanager-webhook"
        }
    }
}
```
5. Use [Zappa](https://github.com/Zappa/Zappa) to deploy your Webhook
to AWS Lambda (this is installed as part of the dependencies above):
```bash
zappa deploy
```
6. Take note of the URL that is returned by the `zappa deploy` command,
eg. `https://3e602e00.execute-api.ap-southeast-3.amazonaws.com/alertmanager`
   (obviously use your own and don't copy and paste this one, or your
Webhook will not work).

**NOTE:** If you get the following error when running the `zappa deploy` command:

<pre>
botocore.exceptions.ClientError:
An error occurred (IllegalLocationConstraintException) when calling
the CreateBucket operation: The unspecified location constraint
is incompatible for the region specific endpoint this request was sent to.
</pre>

This error usually means that your S3 bucket name is not unique, and that you
should change it to something different, since the S3 bucket names are not
namespaced and are global for everyone.

7. Check the status of the API Gateway URL that was created by zappa:
```bash
zappa status
```
8. Test your webhook by making a curl request to the URL that was returned
by `zappa deploy`:
```
curl https://3e602e00.execute-api.ap-southeast-3.amazonaws.com/alertmanager
```
You should expect the following response:
```json
{"status":"ok"}
```
9. Update your Webhook URL in Alertmanager to the one returned by the
`zappa deploy` command.
10. You can view your logs by running:
```bash
zappa tail
```


