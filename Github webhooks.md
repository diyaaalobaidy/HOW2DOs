## First install webhooks
[https://thelinuxnotes.com/index.php/how-to-automatically-deploy-from-github-to-server-using-webhook/](useful link)
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install webhook
```
## Edit the config file to match your code
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
## Edit the script you want, for example:
pull.sh
```bash
#!/bin/bash
git pull
php artisan optimize
```
## Make a service file to run the webhook service
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
## Make NGINX to have a url for the webhook service to be secure
server {
	server_name webhooks.mally.app;

	location / {
		include proxy_params;
		proxy_pass http://localhost:9000;
	}
}
## You can add ssl certificate, [https://github.com/diyaaalobaidy/HOW2DOs/blob/main/Deploy%20on%20nginx.md](can be found here)
