Just a bit of documentation on how I managed to setup a fully functional email server on my pi.
Prenote - This setup works seemlessly with my previously setup apache server with it's own domain powered by cloudflare.

My email server is comprised of 4 programs:
  Postfix - sends/recieves emails
  Dovecot - POP/IMAP management [authentication, basically]
  Roundcube - Web app
  MariaDB - database for users and emails
  OpenDKIM - Signs outgoing emails with a private key
  Certbot - Obtain a free SSL certificate from Let's Encrypt
  Apache - Web server backend
  PHP - Bi-directional communication (roundcube dependency)
  
Obtaining a domain:
First of all, you need to have a domain connected to DNS service - I would recommend using cloudflare and, if you are
unwilling to splurge on a paid domain, obtaining a domain from dot.tk

Zone configuration:
An email (and web) server require specific dns records, the most basic of setups will have the following records:
A - point the domain to your ip
CName - "subdomain" where your web client will be accessed
MX - required** to resolve email requests to your ip

