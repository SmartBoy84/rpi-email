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
```smtpd_helo_restrictions = permit_mynetworks, permit_sasl_authenticated, reject_invalid_helo_hostname,   reject_non_fqdn_helo_hostname, reject_unknown_helo_hostname, check_helo_access hash:/etc/postfix/helo_access```  
  
Create helo_access file in ```/etc/postfix/helo_access``` and add the following to it  
```  
X.X.X.X   REJECT  
domain.com   REJECT  
smtp.domain.com   REJECT  
mail.domain.com   REJECT  
```  
Create database for helo file: ```postmap helo_access```  
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
Certbot by default relies on http verification which means that you need to have port 80 exposed (port forwarded)
If this is not feasible, then another option is to ```certbot --manual --preferred-challenges dns certonly -d host.domain.com```. (Certbot will prompt you to add a DNS TEXT record)

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

Add the following to /etc/apache2/sites-available/domain.com  
```  
<VirtualHost *:443>  
ServerName mail.gabba.ga  
DocumentRoot /var/www/roundcube  
SSLEngine on  
    SSLCertificateFile /etc/apache2/ssl/gabba.ga.pem  
    SSLCertificateKeyFile /etc/apache2/ssl/gabba.ga.key  
  
  
<Directory /var/www/roundcube/>  
  Options +FollowSymLinks  
  # This is needed to parse /var/lib/roundcube/.htaccess. See its  
  # content before setting AllowOverride to None.  
  AllowOverride All  
  <IfVersion >= 2.3>  
    Require all granted  
  </IfVersion>  
  <IfVersion < 2.3>  
    Order allow,deny  
    Allow from all  
  </IfVersion>  
</Directory>  
  
# Protecting basic directories:  
<Directory /var/www/roundcube/config>  
        Options -FollowSymLinks  
        AllowOverride None  
</Directory>  
  
<Directory /var/www/roundcube/temp>  
        Options -FollowSymLinks  
        AllowOverride None  
        <IfVersion >= 2.3>  
          Require all denied  
        </IfVersion>  
        <IfVersion < 2.3>  
          Order allow,deny  
          Deny from all  
        </IfVersion>  
</Directory>  
  
<Directory /var/www/roundcube/logs>  
        Options -FollowSymLinks  
        AllowOverride None  
        <IfVersion >= 2.3>  
          Require all denied  
        </IfVersion>  
        <IfVersion < 2.3>  
          Order allow,deny  
          Deny from all  
        </IfVersion>  
</Directory>  
</VirtualHost>  
```  
Then remove the default site: ```rm 000*```  
Activate new site  
```  
a2ensite domain.com  
Sudo service apache2 restart  
```  
  
Install roundcube by going to mail.domain.com/installer  
  
Follow: https://www.linode.com/docs/email/postfix/configure-spf-and-dkim-in-postfix-on-debian-8/  
To stop emails from going to spam folder  
You also need to set up a PTR record and add "smtp_tls_security_level = may" to encrypt outbound messages (in main.cf)
To get green padlock in gmail (both way encryption) do this https://www.infopackets.com/news/10218/how-fix-send-encrypted-email-gmail-postfix-tls-ssl-certificates

I was not able to receive emails from GMail and looking at the logs told me that it was because postfix was not using TLS. This was because of mismatching certificates. I solved this by doing the following:  
First check if you have the same problem by doing:  
```  
sudo -i  
(openssl x509 -noout -modulus -in /etc/ssl/certs/ssl-cert-snakeoil.pem | openssl md5 ; openssl rsa -noout -modulus -in /etc/ssl/private/ssl-cert-snakeoil.key | openssl md5) | uniq  
```  
If this command results in two different hashes for the .key .pem file it means the certificates are not matching and you have to regenerate them which you can do by using this command:  
```sudo make-ssl-cert generate-default-snakeoil --force-overwrite```  
Restart postfix: ```sudo service postfix restart```  
  
VOILA! You now have a fully-featured, fully-functioning email server running on your pi!  
  
Sidenotes:  
Ensure that you update your ip on cloudflare whenever it changes  
It's obvious but in case you didn't now, you won't receive any emails if the pi is offline - there isn't some offshort reservoir that keeps your emails until you turn the pi back on! ;=)  
You need to regularly refresh your lets-encrypt certificates with the same command you used to create them (expiry is 90 dates I think - crontab?)
  
Ok, so I am following this guide now to reinstall a mail server on a fresh install but several things are broken:

Wup, who'da thoght you need php to get an application written in php to work, follow this guide:
https://www.howtoforge.com/tutorial/ubuntu-roundcube-latest/#configure-apache-for-roundcube
They also have a smaller config for apache

The other problems you'll encounter will be with dovecot, just do service status dovecot to troubleshoot gl

MAJOR PROBLEM: WHEN CREATING CERTIFICATES FOR DOVECOT MAKE SURE YOU MAKE THEM FOR THE UNPROXIED DNS RECORD (NOT domain.com OR mail.domain.com) AND THIS REQUIRED PORT 80 TO BE OPENED FOR CERTBOT TO VERIFY
Any other problems you encounter will likely be due to you following steps from other sites concurrently or skipping over steps, I can verify that following this guide step by step will work
  
Bye.  
