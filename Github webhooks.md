# First install webhooks
[https://thelinuxnotes.com/index.php/how-to-automatically-deploy-from-github-to-server-using-webhook/](useful link)
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install webhook
'''
/etc/webhook.conf
```json
[
        {
                "id": "{id for the pull}",
                "execute-command": "{deploy.sh}",
                "command-working-directory": "{project-root}",
                "response-message": "Executing deploy script...",
                "trigger-rule": {
                        "match": {
                                "type": "payload-hash-sha1",
                                "secret": "{the secret key}",
                                "parameter": {
                                        "source": "header",
                                        "name": "X-Hub-Signature"
                                }
                        }
                }
        }
]
```

pull.sh
```bash
#!/bin/bash
git pull
php artisan optimize
```

/lib/systemd/system/webhook.service
```bash
[Unit]
Description=Small server for creating HTTP endpoints (hooks)
Documentation=https://github.com/adnanh/webhook/
ConditionPathExists=/etc/webhook.conf

[Service]
ExecStart=/usr/bin/webhook -nopanic -hooks /etc/webhook.conf

[Install]
WantedBy=multi-user.target
```
