## Linux Server Configuration

In this project we take the [Item Catalog](https://github.com/pekerow/item-catalog) application and place it onto a Ubuntu Linux web server instance using [Linode](https://www.linode.com/) with public key encryption.  We obtain an SSL Certificate and reconfigure [Apache HTTP Server](http://httpd.apache.org/) to serve the web app via HTTPS instead of HTTP. The Object-Relational Mapping tool [SQLAlchemy](https://www.sqlalchemy.org/) turns database contents into Python objects, which are delivered to the app by [Flask](http://flask.pocoo.org/). The database engine itself is [PostgreSQL](https://www.postgresql.org/).

The application is accessible via my personal URL at https://www.pekerow.site. The application code differs only slightly from the original [Item Catalog](https://github.com/pekerow/item-catalog) code and can be viewed separately in the [Catalog](https://github.com/pekerow/catalog) repository. This repo exists solely to document how we bring an application from the development environment to production. The choice of Linode is a personal preference. As most of the work is done in the terminal, the same could be done at AWS, DigitalOcean or any competing provider.


### First steps
We first create a Linode account and initialize a Ubuntu 22.04 LTS instance. Linode provides its own browser-based "Lish" console for use at the outset. We use this to login as ``root``, where our first step will be to install updates by typing ``apt-get update`` followed by ``apt-get upgrade``. Since we are professionals here, we now want to set up our instance so we can log in using an actual terminal program such as zsh. This requires creating our own public and private key, which we do on our local machine to prevent key theft. Proceed as follows:

1. On our local terminal, type ``ssh-keygen`` to create the new key pair. Unless you specify otherwise a pair of keys will be made withe default name ``id_rsa`` in the default ``.ssh`` directory on your machine, usually directly under ``/home/``. 
2. Type ``cat <filepath>/.ssh/id_rsa.pub`` and then copy the public key in its entirety, which will be several lines long. 
3. On our instance, navigate to the ``.ssh`` directory , type ``nano authorized_keys`` and then paste your public key into the empty file and save it.
4. Return to /home/ with ``cd ..`` and then change the ``.ssh`` directory file permissions by typing ``chmod 700 .ssh``
5. Change the ``authorized_keys`` permissions by typing ``chmod 600 .ssh/authorized_keys``
6. (Optional) On your local machine, you may move the private key file ``id_rsa`` to a convenient directory on your machine from which you will log in to your instance. If you didn't specify a different name when creating the key pair, you may rename the file. DO NOT change the extension or the file's contents. You can also leave the file where it is, but you will need to specify the private key's filepath when logging in. 
NOTE: If you are a Mac user, it may be necessary to change permissions on your private key file to ``chmod 400``  Otherwise a warning may appear about permissions being too open, and the key will be disallowed.


### sshd configuration

Best security practices dictate that we disable the ``root`` login and create a new user with sudo access privileges. We should also open port 2200 to use in place of the usual ssh port 22 (to be closed).

1. Type ``adduser username`` and ``usermod -aG sudo username``, substituting  ``username`` with a name of your choice.
2. Go into the sshd configuration file by typing ``nano /etc/ssh/sshd_config``
3. Change ``PermitRootLogin prohibit password`` to ``PermitRootLogin no``
4. Change ``PasswordAuthentication yes`` to ``PasswordAuthentication no``
5. Open TCP port 2200 in your Linode instance [firewall](https://cloud.linode.com/firewalls).
6. Restart ssh by typing ``service ssh restart``
7. Log out of the instance. To log back in using the new username you created, navigate to the private key location on your local machine and type ``ssh -i isd_rsa@45.56.107.121 -p 2200`` (substitute "id_rsa" with the name of your private key and my IP address with the one for your instance). Your new user will have all the access privileges of the root user when prefacing commands with ``sudo``. 


### UFW (uncomplicated firewall) configuration

To ensure proper access to our application, we must open the correct ports and disable the default ssh port 22, which is a common target for bad actors:

1. Deny all incoming requests: ``sudo ufw default deny incoming``
2. Allow all outgoing requests: ``sudo ufw default allow outgoing``
3. Allow ssh: ``sudo ufw allow ssh``
4. Open TCP port 2200 (again): ``sudo ufw allow 2200/tcp``
5. Enable http (needed for Apache server): ``sudo ufw allow www``
6. Enable https: ``sudo ufw allow 443/tcp``
7. Enable udp: ``sudo ufw allow 123/udp``
8. Disable port 22: ``sduo ufw deny 22/tcp``
9. Activate firewall: ``sudo ufw enable``

These settings can also be activated in the Linode [firewall](https://cloud.linode.com/firewalls) settings in lieu of using UFW. Since we are configuring our server with Linux via a terminal, UFW is preferable, allowing us to view our firewall settings in the same context.


### Python and package setup 

Now it's time to insert our application code by cloning the [Catalog](https://github.com/pekerow/catalog) repository. Python 3.10 is installed by default. In line with best practices we first want to create a virtual Python environment:

1. Navigate to ``/var/www/`` and ``mkdir catalog``
2. Install ``sudo apt-get install python3-venv``
3. Create: ``python3 -m venv venv``
4. Activate: ``source venv/bin/activate``
5. Clone: ``git clone https://github.com/pekerow/catalog`` (after setting git up on your instance)

Now we can install all necessary packages:

* ``sudo apt-get python-pip``
* ``pip install psycopg2``
* ``pip install flask`` 
* ``pip install Flask-SQLAlchemy`` 
* ``pip install requests`` 
* ``pip install httplib2``
* ``pip install oauth2client``
* ``sudo apt-get postgresql``

Be sure to pay attention to any conflicts or prerequisites that may arise and Google accordingly to resolve them.


### Postgresql setup

The development version of [Item Catalog](https://github.com/pekerow/item-catalog) used SQLite to implement the database scheme. In the production environment we switch it out for the more robust and secure Postgresql engine. We must first manually configure the databse:

1. Log into default ``postgres`` user: ``sudo su - postgres``
2. Open the database: ``psql``
3. Create new db user catalog: ``CREATE USER catalog;`` (will be invoked by application code)
4. Create password: ``ALTER USER catalog with PASSWORD 'catalog';`` (single quotes only)
5. Create database: ``CREATE DATABASE catalog WITH OWNER catalog;``
6. Revoke public access: ``REVOKE ALL ON SCHEMA public FROM public;``
7. Give catalog user admin privileges: ``GRANT ALL ON schema public TO catalog;``

A special user named ``catalog`` has been created to effect proper creation of the database. Only the ``catalog`` user has power over the database; all public access has been revoked. Note that Postgresql requires that statements end with semicolons as in C-style languages (C++, Java, C# and Javascript).


### Apache and wsgi configuration

1. Install Apache: ``sudo apt-get install apache2``
2. Install mod-wsgi: ``sudo apt-get install libapache2-mod-wsgi python-dev``
3. Enable the module: ``sudo a2enmod wsgi``
4. Restart Apache: ``sudo service apache2 restart``

Web Server Gateway Interface (WSGI) is used for calling the application code in cojunction with the Apache server:

5. Create the wsgi file with ``sudo nano catalog.wsgi`` and paste:

```
import sys
import logging

logging.basicConfig(stream=sys.stderr)
sys.path.append('/var/www/catalog/venv/lib/python3.10/site-packages')
sys.path.insert(0, '/var/www/')

from catalog import app as application
application.secret_key = 'super_secret_key'
```

6. Navigate to ``/etc/apache2/sites-available`` and type ``sudo nano catalog.conf``. Paste the following and save:

```
<VirtualHost *:80>
      ServerName www.pekerow.site
      ServerAdmin pekerow@gmail.com
      WSGIScriptAlias / /var/www/catalog/catalog.wsgi
      <Directory /var/www/catalog/>
          Order allow,deny
          Allow from all
      </Directory>
</VirtualHost>

<VirtualHost *:443>
        DocumentRoot /var/www/catalog
        ServerName www.pekerow.site
        ServerAdmin pekerow@gmail.com
        WSGIScriptAlias / /var/www/catalog/catalog.wsgi
        <Directory /var/www/catalog/>
                Order allow,deny
                Allow from all
        </Directory>
</VirtualHost>
```

7. Disable default Apache site: ``sudo a2dissite 000-default``
8. Enable catalog app: ``sudo a2ensite catalog``
9. Reload and restart Apache: ``sudo service apache2 reload`` and ``sudo service apache2 restart``

The ``catalog.conf`` file enables Apache to locate and serve the Catalog app on both http (port 80) and https (port 443). Note that the ServerName is not a linode.com or IP address. For SSL certificate purposes, you must obtain a real domain name from a domain registrar and then configure it to point to your Linode site. This is easy enough, but we aren't done yet...


### Obtain SSL certificate

We can't just tell Apache to serve https--we must first acquire an actual SSL certificate so the web browser can verify that our site is actually secure. It cannot be an ad-hoc, self-signed certificate or the browser will reject the connection. Fortunately, we can acquire one for free using the [Certbot](https://certbot.eff.org/instructions?ws=nginx&os=ubuntufocal) command-line utility and it will configure the cetificate automatically. Having already configured our ``catalog.conf`` file, we can simply follow Certbot's self-explanatory instructions to obtain the certificate and have it working with our app in just a couple of minutes! Certbot will add some additional lines of code to ``catalog.conf`` with the revised file looking like this:

```
VirtualHost *:80>
      ServerName www.pekerow.site
      ServerAdmin pekerow@gmail.com
      WSGIScriptAlias / /var/www/catalog/catalog.wsgi
      <Directory /var/www/catalog/>
          Order allow,deny
          Allow from all
      </Directory>
      ErrorLog ${APACHE_LOG_DIR}/error.log
      LogLevel warn
      CustomLog ${APACHE_LOG_DIR}/access.log combined
      RewriteEngine on
      RewriteCond %{SERVER_NAME} =www.pekerow.site
      RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
</VirtualHost>
<VirtualHost *:443>
        DocumentRoot /var/www/catalog
        ServerName www.pekerow.site
        ServerAdmin pekerow@gmail.com
        WSGIScriptAlias / /var/www/catalog/catalog.wsgi
        <Directory /var/www/catalog/>
                Order allow,deny
                Allow from all
        </Directory>
        ErrorLog ${APACHE_LOG_DIR}/error.log
        LogLevel warn
        CustomLog ${APACHE_LOG_DIR}/access.log combined
        SSLEngine on
        Include /etc/letsencrypt/options-ssl-apache.conf
        SSLCertificateFile /etc/letsencrypt/live/www.pekerow.site/fullchain.pem
        SSLCertificateKeyFile /etc/letsencrypt/live/www.pekerow.site/privkey.pem
</VirtualHost>
```

These instuctions redirect our Apache server to serve https instead of the default http. Certbot creates the path ``/etc/letsencrypt/live/``to store the certificate and associated key, allowing the client browser to authenticate the certificate. It also creates an error log for debugging at ``/var/log/apache2/error.log`` and an access log for vieweing user-driven events at ``/var/log/apache2/access.log``


### Reconfigure OAuth
In the Google Cloud Console, altered the application's credentials section: 
* Added ``https://www.pekerow.site`` to Authorized Javscript Origins
* Added ``https://www.pekerow.site/login`` ``https://www.pekerow.site/gconnect`` and ``https://www.pekerow.site/oauth2callback`` to Authorized Redirect URIs
* Downloaed the new ``client_secrets.json`` file and replaced it on the server to ensure proper routing and authentication of users.


### Sources Consulted

* https://varunpant.com/posts/how-to-make-https-requests-with-python-httplib2-ssl/ 
* https://supporthost.com/invalid-ssl-certificate 
* https://www.digicert.com/kb/csr-ssl-installation/apache-openssl.htm 
* https://stackoverflow.com/questions/5257974/how-to-install-mod-ssl-for-apache-httpd
* https://www.ssls.com/knowledgebase/how-to-install-an-ssl-certificate-on-apache/
* https://www.rosehosting.com/blog/how-to-generate-a-self-signed-ssl-certificate-on-linux/
* https://medium.com/techfront/step-by-step-visual-guide-on-deploying-a-flask-application-on-aws-ec2-8e3e8b82c4f7
* https://sectigo.com/resource-library/install-certificates-apache-open-ssl 
* https://supporthost.com/invalid-ssl-certificate
* https://unix.stackexchange.com/questions/155150/where-in-apache-2-do-you-set-the-servername-directive-globally
* https://github.com/daphnetull/linux-sever-config
* https://stackoverflow.com/questions/7210011/amazon-ec2-ssh-timeout-due-inactivity
* https://stackoverflow.com/questions/29933918/ssh-key-permissions-0644-for-id-rsa-pub-are-too-open-on-mac
* https://www.linode.com/docs/security/firewalls/configure-firewall-with-ufw/
* https://www.digitalocean.com/community/tutorials/ufw-essentials-common-firewall-rules-and-commands
* https://www.youtube.com/watch?v=-J9wUW5NhOQ
* https://www.liquidweb.com/kb/changing-the-ssh-port/
* https://askubuntu.com/questions/147241/execute-sudo-without-password
* https://medium.com/coding-blocks/creating-user-database-and-adding-access-on-postgresql-8bfcd2f4a91e
* https://serverfault.com/questions/520195/how-does-servername-and-serveralias-work
* https://askubuntu.com/questions/391184/but-is-referred-to-by-another-package-finding-that-package
* https://www.a2hosting.com/kb/developer-corner/apache-web-server/viewing-apache-log-files
* https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps
