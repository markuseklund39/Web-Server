# Web-Server
The purpose of this project was to set up a web server on a computer running Linux Mint, using a domain name that I already owned. The web server was required to host a download page protected by password authentication.

## Web Server Setup

I installed and enabled Apache2. The command
systemctl start apache2 starts the Apache service, and
systemctl enable apache2 ensures that it starts automatically on system boot.
Apache creates a default webpage on localhost, located in /var/www/html/.

The firewall UFW was configured to allow HTTP traffic using the command:
ufw allow 'Apache'.

## Domain Configuration

I logged in to my domain registrar and created an A record that points mydomain.se to my public IPv4 address.

Next, I created a virtual host configuration file for the domain at
/etc/apache2/sites-available/mydomain.se.conf with the following content:

<VirtualHost *:80>
    ServerName mydomain.se
    ServerAlias www.mydomain.se

    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/mydomain.se

    <Directory /var/www/mydomain.se>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/mydomain.se-error.log
    CustomLog ${APACHE_LOG_DIR}/mydomain.se-access.log combined
</VirtualHost>


I then created the directory /var/www/mydomain.se and added an index.html file.
The site was enabled using a2ensite mydomain.se.conf, followed by
systemctl reload apache2.

## Cloudflare Tunnel

Since my Internet Service Provider does not allow hosting a web server directly on my public IP address, I chose to use a Cloudflare Tunnel.

I downloaded and installed cloudflared, logged in using
cloudflared tunnel login, created an account, and added my domain.
A certificate was downloaded automatically. Cloudflare provided new name servers, which I updated at my domain registrar.

I then created a tunnel using the following command:

cloudflared tunnel --origincert ~/.cloudflared/cert.pem create min-tunnel


A Cloudflare configuration file was created at
~/.cloudflared/config.yml with the following content:

tunnel: <TUNNEL-ID>
credentials-file: /home/user/.cloudflared/<TUNNEL-ID>.json

ingress:
  - hostname: mydomain.se
    service: http://localhost:80
  - hostname: www.mydomain.se
    service: http://localhost:80
  - service: http_status:404


DNS records were created using the commands:

cloudflared tunnel route dns min-tunnel mydomain.se
cloudflared tunnel route dns min-tunnel www.mydomain.se


The tunnel was started with:

cloudflared tunnel run min-tunnel


At this point, the website was successfully accessible from the internet.

## Password-Protected Download Page

A download directory was created at
/var/www/mydomain.se/downloads.

I also created a directory for password files at
/etc/apache2/.htpasswd.
User credentials were generated using the command:

htpasswd -c user guest


Inside the downloads directory, I created a .htaccess file with the following content:

AuthType Basic
AuthName "downloads"
AuthUserFile /etc/apache2/.htpasswd/user
Require valid-user


Finally, I set the correct permissions:

chmod 640 /etc/apache2/.htpasswd/user
chown root:www-data /etc/apache2/.htpasswd/user
chown -R www-data:www-data /var/www/mydomain.se/downloads


After restarting Apache, the downloads directory was successfully protected by password authentication.
