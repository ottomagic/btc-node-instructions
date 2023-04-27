# Configuring a disk space email alert

Some reading material:
- https://fedingo.com/shell-script-to-check-disk-space-and-send-email-alerts/
- https://linuxhandbook.com/linux-send-email-ssmtp/
- https://help.ubuntu.com/community/EmailAlerts

Future reading:
https://www.digitalocean.com/community/tutorials/how-to-monitor-server-health-with-checkmk-on-ubuntu-20-04

## Install and configure ssmtp for sending emails

Run the following command to install ssmtp:
```bash
sudo apt install ssmtp
```

Edit the configuration file:
```bash
sudo -e /etc/ssmtp/ssmtp.conf
```
Example content for configuration:
```
#
# Config file for sSMTP sendmail
#
# The person who gets all mail for userids < 1000
# Make this empty to disable rewriting.
Root=noreply@ottomagic.dev

# The place where the mail goes. The actual machine name is required no
# MX records are consulted. Commonly mailhosts are named mail.domain.com
Mailhub=mail.privateemail.com:465

# Where will the mail seem to come from?
rewriteDomain=ottomagic.dev

AuthUser=SMTP_USERNAME
AuthPass=SMTP_PASSWORD

# The full hostname
Hostname=vmd78845.contaboserver.net

# Are users allowed to set their own From: address?
# YES - Allow the user to specify their own From: address
# NO - Use the system generated From: address
FromLineOverride=YES
UseTLS=Yes
```

Edit the configuration for reverse aliases:
```bash
sudo nano /etc/ssmtp/revaliases
```
Example config:
```
# sSMTP aliases
#
# Format:       local_account:outgoing_address:mailhub
#
# Example: root:your_login@your.domain:mailhub.your.domain[:port]
# where [:port] is an optional port number that defaults to 25.
root:noreply@ottomagic.dev:mail.privateemail.com:465
admin:noreply@ottomagic.dev:mail.privateemail.com:465
```

Create a script to check disk space and send an email alert if space is low:
```
sudo nano /home/admin/check_disk.sh
```
Script content:
```bash
#!/bin/bash

CURRENT=$(df / | grep / | awk '{ print $5}' | sed 's/%//g')
THRESHOLD=95

if [ "$CURRENT" -gt "$THRESHOLD" ] ; then

/usr/sbin/ssmtp -t  << EOF
To: otto@ottomagic.me
From: noreply@ottomagic.dev
Subject: Disk Space Alert: BTC node

Your root partition remaining free space is critically low. Used: $CURRENT%
EOF

fi
```

Set the file as executable:
```bash
sudo chmod +x check_disk.sh
```

```bash
crontab -e
```

Add the following lines:
```
MAILTO=otto@ottomagic.me
MAILFROM=noreply@ottomagic.dev    (NOTE! Ubuntu cron currently does not support this! Maybe switches to "cronie" implementation in the future)
CRON_TZ=Europe/Berlin

@daily /home/admin/check_disk.sh > /dev/null
```
