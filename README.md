Just a bit of documentation on how I managed to setup a fully functional email server on my pi.
Prenote - This setup works seemlessly with my previously setup apache server with it's own domain powered by cloudflare.

My email server is comprised of 4 programs:
•	Postfix - sends/recieves emails
•	Dovecot - POP/IMAP management [authentication, basically]
•	Roundcube - Web app
•	MariaDB - database for users and emails
•	OpenDKIM - Signs outgoing emails with a private key
•	Certbot - Obtain a free SSL certificate from Let's Encrypt
•	Apache - Web server backend
•	PHP - Bi-directional communication (roundcube dependency)
  
Obtaining a domain:
First of all, you need to have a domain connected to DNS service - I would recommend using cloudflare and, if you are unwilling to splurge on a paid domain, obtaining a domain from dot.tk

Zone configuration:
An email (and web) server require specific dns records, the most basic of setups will have the following records:
A - point the domain to your ip
CName - "subdomain" where your web client will be accessed
MX - required to resolve email requests to your ip
 
*This is the config I used but you do not need the first A record not the www

Postfix setup
Install the package: ```sudo apt-get install postfix```
During the installation you will have be asked to specify two things: 
•	The general type of mail configuration: Internet site 
•	System mail name: domain.com

Now we edit the generating config files:
In /etc/postfix/main.cf
•	Set ```inet_protocols``` to ipv4 instead of all [```inet_protocols = ipv4```]
•	Set ```myhostname``` to your domain [```myhostname = domain.com```]
Add the following lines to the bottom of the file:
•	home_mailbox = Maildir/ 
•	mailbox_command =

If you are on a local host, it is extremely likely that your ISP block outbound requests from port 25 so you will have to setup a custom smtp relay. 
To see if it’s blocked type the following in a linux terminal (or enable telnet in windows)
•	```telnet smtp.gmail.com```
It should say “trying [ip]…” and respond resulting in more text within seconds. If it doesn’t then the post is most likely blocked. Once you find the relay do the following:
•	Set the relayhost to smtp.yourprovider.com [relayhost = smtp.yourprovider.com]
Restart postfix to have this config take effect [```sudo service postfix restart```]
Test if the config worked by telnetting into localhost on port 25 and running the following string of commands in series:
•	Ehlo
•	Mail from: you@domain.com
•	Rcpt to: user@mail.com
•	Data
•	Subject: test
•	Test
•	.
•	Quit
*These commands sequence will create an email and send it to user@mail.com (external email address)

Dovecot
Install: ```sudo apt-get install dovecot-common dovecot-imapd```
Create folders in template dir and setup mail dir
•	sudo maildirmake.dovecot /etc/skel/Maildir 
•	sudo maildirmake.dovecot /etc/skel/Maildir/.Drafts 
•	sudo maildirmake.dovecot /etc/skel/Maildir/.Sent 
•	sudo maildirmake.dovecot /etc/skel/Maildir/.Spam 
•	sudo maildirmake.dovecot /etc/skel/Maildir/.Trash 
•	sudo maildirmake.dovecot /etc/skel/Maildir/.Templates
•	sudo cp -r /etc/skel/Maildir 
•	/home/pi/ sudo chown -R pi:pi 
•	/home/pi/Maildir sudo chmod -R 700 /home/pi/Maildir


Securing the mail server
Add the following the bottom of the main.cf file:
```smtpd_helo_restrictions = permit_mynetworks, permit_sasl_authenticated, reject_invalid_helo_hostname, reject_non_fqdn_helo_hostname, reject_unknown_helo_hostname, check_helo_access hash:/etc/postfix/helo_access```

Create helo_access file in ```/etc/postfix/helo_access``` and add the following to it
•	X.X.X.X   REJECT
•	domain.com   REJECT
•	smtp.domain.com   REJECT
•	mail.domain.com   REJECT
*replace x.x.x.x with your ip
Restart: ``` sudo service postfix restart```

Configure Dovecot
In ``` /etc/dovecot/dovecot.conf``` do the following
•	Replace ```#listen = *, ::``` with ```listen = *```
In ``` /etc/dovecot/conf.d/10-mail.conf```
•	Set ```mail_location``` to ``` maildir:~/Maildir```
In ``` /etc/dovecot/conf.d/10-master.conf```
•	Comment all the lines from default service auth paragraph
•	Add the following lines to the end of the file:
```
service auth {
        unix_listener /var/spool/postfix/private/auth {
                mode = 0660
                user = postfix
                group = postfix
        }
}
```

In 
```/etc/dovecot/conf.d/10-auth.conf```
•	Uncomment and set ```disable_plaintext_auth``` to no
In main.cf [postfix] add the following:
•	smtpd_sasl_type = dovecot
•	smtpd_sasl_path = private/auth
•	smtpd_sasl_auth_enable = yes

Restart both postfix and dovecot: 
```
sudo service postfix restart 
sudo service dovecot restart
```

Generate ssl cert and enable IMAPS
Get certbot: ```sudo apt install certbot``` 
Make certificate: ```sudo certbot certonly --standalone -d host.domain.com ```
*ensure that the hostname matches the one that acts as your dns (unclouded) not just domain.com! Dovecot will fail to start otherwise.
In ```/etc/dovecot/conf.d/10-master.conf``` ensure the config looks like the following
```
service imap-login {
  inet_listener imap {
    port = 143
  } 
  inet_listener imaps {
    port = 993
    ssl = yes
  }
}
```

In ```/etc/dovecot/conf.d/10-ssl.conf``` set SSL to yes
Uncomment and set the certificate locations as followed:
```
ssl_cert = </etc/letsencrypt/live/host.domain.com /fullchain.pem
ssl_key = </etc/letsencrypt/live/ host.domain.com /privkey.pem
```
Restart dovecot: ``` sudo service dovecot restart```

Setup Roundcube and MySQL server
Get mariadb: ```sudo apt-get install mariadb-server```
Run ``` mysql_secure_installation``` to setup the database – follow the instructions
Create roundcube user and database:
```
sudo mysql -uroot
CREATE USER 'roundcube'@'localhost' IDENTIFIED BY 'password'; [set password to whatever you wish]
CREATE DATABASE roundcubemail;
GRANT ALL PRIVILEGES ON roundcubemail.* to 'roundcube'@'localhost';
FLUSH PRIVILEGES;
Quit
```

Install roundcube:
Get dependencies:
``` sudo apt-get install apache2 mariadb-server php7.2 php7.2-gd php-mysql php7.2-curl php7.2-zip php7.2-ldap php7.2-mbstring php-imagick php7.2-intl php7.2-xml unzip wget curl -y
```wget https://github.com/roundcube/roundcubemail/releases/download/1.4.9/roundcubemail-1.4.9-complete.tar.gz```
tar xfz roundcubemail-*.tar.gz
mv -r [roundcube folder] /var/www/roundcube
```

At this point setup roundcube in apache config however you want but I’ll tell you how I did it


