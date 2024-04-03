# Email Server

Useful links: 
- [Install postfix](https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-postfix-on-ubuntu-20-04)
- [Install dovecot](https://ubuntu.com/server/docs/mail-dovecot)
- [Install opendkim](https://tecadmin.net/setup-dkim-with-postfix-on-ubuntu-debian/)

## Postfix

- Install postfix

```bash
sudo apt install postfix
```

- Edit configuration files

```bash
sudo nano /etc/postfix/main.cf
```

Write the following configuration to the file, **don't forget to replace *example.com* with your domain name**

```bash

smtpd_banner = $myhostname ESMTP $mail_name (Ubuntu)
biff = no
append_dot_mydomain = no
readme_directory = no
compatibility_level = 3.6
smtpd_tls_cert_file=/etc/letsencrypt/live/example.com/fullchain.pem
smtpd_tls_key_file=/etc/letsencrypt/live/example.com/privkey.pem
smtpd_tls_security_level=may
smtpd_use_tls = yes
smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_scache
smtpd_tls_protocols = !SSLv2, !SSLv3
smtp_tls_cert_file=/etc/letsencrypt/live/example.com/fullchain.pem
smtp_tls_key_file=/etc/letsencrypt/live/example.com/privkey.pem
smtp_tls_security_level=may
smtp_use_tls = yes
smtp_tls_CApath=/etc/ssl/certs
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes
smtpd_sasl_security_options = noanonymous
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtpd_relay_restrictions = permit_mynetworks permit_sasl_authenticated defer_unauth_destination
myhostname = example.com
alias_maps = hash:/etc/aliases
alias_database = hash:/etc/aliases
myorigin = /etc/mailname
mydestination = $myhostname, example.com, localhost.localdomain, localhost
relayhost =
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128
mailbox_size_limit = 0
recipient_delimiter = +
inet_interfaces = all
inet_protocols = ipv4
default_privs = mail
home_mailbox = Maildir/
milter_default_action = accept
milter_protocol = 6
smtpd_milters = local:opendkim/opendkim.sock
non_smtpd_milters = $smtpd_milters
```

Open the master.cf file
```bash
nano /etc/postfix/master.cf
```

Uncomment the following line and save the file

```bash
submission inet n       -       y       -       -       smtpd
```

**Note**: You need certbot for the certificate, see [this tutorial](https://github.com/diyaaalobaidy/QAF-HOW2DOs/blob/main/Deploy%20on%20nginx.md) for more details

**Note**: If you only want to send emails, you don't need Dovecot and Roundcube, skip to opendkim

---
## Dovecot

- Install dovecot

```bash
sudo apt install dovecot-imapd dovecot-pop3d
```

- Edit configuration files

```bash
sudo nano /etc/dovecot/dovecot.conf
```

Write the following configuration to the file, **don't forget to replace *example.com* with your domain name**

```bash
first_valid_uid = 8
mail_uid = 8
mail_gid = mail
disable_plaintext_auth = no
mail_privileged_group = mail
mail_location = maildir:/home/%n/Maildir

protocols = "imap"


auth_mechanisms = plain

passdb {
  driver = passwd-file
  args = scheme=PLAIN username_format=%u /etc/dovecot/users
}
userdb {
  driver = passwd-file
  args = username_format=%u /etc/dovecot/users
}

namespace main {
  type = private
  separator = /
  prefix =
  inbox = yes
  location = maildir:/home/%n/Maildir
}

service auth {
  unix_listener /var/spool/postfix/private/auth {
    group = postfix
    mode = 0660
    user = postfix
  }
}

ssl=required
ssl_cert = </etc/letsencrypt/live/example.com/fullchain.pem
ssl_key = </etc/letsencrypt/live/example.com/privkey.pem
log_path = /var/log/dovecot.log
```

---
## Opendkim

- Install opendkim

```bash
sudo apt install opendkim opendkim-tools
```

- Add the postfix user to the opendkim group

```bash
sudo usermod -a -G opendkim postfix
```

- Generate Public and Private DKIM Keys, **don't forget to replace *example.com* with your domain name**
    - Create a proper directory structure to keep the Key files secure
    
    ```bash
    sudo mkdir -p /etc/opendkim/keys 
    sudo chown -R opendkim:opendkim /etc/opendkim
    sudo chmod  744 /etc/opendkim/keys 
    ```

    - Generate public and private DKIM keys using opendkim-genkey command line utility
    
    ```bash
    sudo mkdir /etc/opendkim/keys/example.com 
    sudo opendkim-genkey -b 2048 -d example.com -D /etc/opendkim/keys/example.com -s default -v o opendkim-genkey -b 2048 -d example.com -D /etc/opendkim/keys/example.com -s default -v 
    ```

    - Set appropriate permissions on the private key file
    
    ```bash
    sudo chown opendkim:opendkim /etc/opendkim/keys/example.com/default.private
    ```

    - Get the value of the public key

    ```bash
    sudo cat /etc/opendkim/keys/example.com/default.txt
    ```
    You will see something like this:

    ```bash
    default._domainkey      IN      TXT     ( "v=DKIM1; h=sha256; k=rsa; "
          "p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAwC/i/W8cVs5610MpSw1DmRWr5Dh7979SBpmSBpdzmKxyRr1S8hwapB2wWypouxS1RP3s9eEW9Oek2eKNAySZUvb6vQgUP+EK5sBuNe/bR4yvyc9pH9+eR2qvEmky4xksSNaS34F74ZUshwV1QSn8eG/5lTrxJD5TUv3/AymqsmOyT5ya9ga0smNtz+3yP9zAbMsGysnVFS2EQN"
          "9fIUc3S7tqpN9FJhcZG7DVfqcMNUDP7q+9cbu/i9UoFmRbuQW3em1JSGFnu0IwRfnmgPvH4dwjLL9DzXkC576RusuFiDjXzgOtTn/KOHUJ1MoF/vp52hwi+QZPPRfF3ILZbe/+0wIDAQAB" )  ; ----- DKIM key default for tecadmin.net
    ```

    - Copy the output value and remove the quotes and spaces, copy from `v=DKIM1` to the end of the key, in this case, we will copy this:
    
    ```bash
    v=DKIM1;h=sha256;k=rsa;p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAwC/i/W8cVs5610MpSw1DmRWr5Dh7979SBpmSBpdzmKxyRr1S8hwapB2wWypouxS1RP3s9eEW9Oek2eKNAySZUvb6vQgUP+EK5sBuNe/bR4yvyc9pH9+eR2qvEmky4xksSNaS34F74ZUshwV1QSn8eG/5lTrxJD5TUv3/AymqsmOyT5ya9ga0smNtz+3yP9zAbMsGysnVFS2EQN9fIUc3S7tqpN9FJhcZG7DVfqcMNUDP7q+9cbu/i9UoFmRbuQW3em1JSGFnu0IwRfnmgPvH4dwjLL9DzXkC576RusuFiDjXzgOtTn/KOHUJ1MoF/vp52hwi+QZPPRfF3ILZbe/+0wIDAQAB
    ```
    - Go to your DNS provider to create the dkim record
    
    - Create a TXT record and name it `default._domainkey`

    - Paste the value you copied from the previous step in the TXT record's content field and save

    - You can test the key to verify that everything is working correctly

    ```bash
    sudo opendkim-testkey -d example.com -s default -vvv 
    ```

    - You should see something like this:

    ```bash
    opendkim-testkey: using default configfile /etc/opendkim.conf
    opendkim-testkey: checking key 'default._domainkey.example.com'
    opendkim-testkey: key not secure
    opendkim-testkey: key OK
    ```

---
## Roundcube

- Install Roundcube

    Unzip the `roundcube.zip` file from the repository in `/var/www`

- Edit config file `/var/www/roundcube/config.inc.php` and replace `example.com` with your domain name

- Add to Nginx server, see [this tutorial](https://github.com/diyaaalobaidy/QAF-HOW2DOs/blob/main/Deploy%20on%20nginx.md) on how to make nginx work with php-fpm.

## Create email users

- Create email creating script file, in the home directory or anywhere you like

```bash
nano create_new_email.sh
```

- Write the following contents to the script file and replace `example.com` with your domain name

```bash
#!/bin/bash

# Check if the script is run with root privileges
if [[ $EUID -ne 0 ]]; then
  echo "This script must be run with root privileges." >&2
  exit 1
fi

# Check if the correct number of arguments is provided
if [ "$#" -ne 2 ]; then
  echo "Usage: $0 <username> <password>"
  exit 1
fi

username="$1"
password="$2"

# Create a new user with the given username
adduser --force-badname "$username"

# Add the user to the 'mail' group
usermod -a -G mail "$username"

# Create a Maildir folder in the user's home directory
mkdir "/home/$username/Maildir"

# Set the appropriate ownership for the Maildir folder
chown "$username":mail "/home/$username/Maildir" -R

# Create a user for login in Dovecot
echo "${username}@example.com:${password}:${username}:mail:/home/${username}/Maildir" >> /etc/dovecot/users

echo "User setup completed for $username."
```

- Make the script executable

```bash
sudo chmod +x create_new_email.sh
```

- Now you can run the script to create a new email account

```bash
./create_new_email.sh <username> <password>
```

**Note**: The username without the `@example.com` because it will be created by the script

- Test by login using roundcube
