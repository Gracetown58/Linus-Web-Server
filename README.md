# Linux Server Configuration
This page explains how to set up a secure linux web server using a Digital
Ocean droplet. This server will then be used to deploy my vinyls catalogue
previously developed.  
The address of the server I deployed is http://104.248.253.240/
## 1. Create the Digital Ocean Droplet
1. On your local machine create RSA key pairs with the following command:  
```
$ ssh-keygen
```
2. [Create digital ocean droplet](https://www.digitalocean.com/docs/droplets/how-to/create/)  
I chose the following configuration: 1 GB Memory / 25 GB Disk / FRA1 - Ubuntu 18.04 x64.  
Provide the ssh `.pub` key you created at the previous step.
3. [Log in to your droplet with ssh](https://www.digitalocean.com/docs/droplets/how-to/connect-with-ssh/):
```  
$ ssh -i /path/to/private/key root@ip.ad.dr.ess
```

## 2. Initial server set up
As explained in this [tutorial](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-18-04), there are a few basic initial configuration steps that should be taken over in order to secure the server:
1. Create a new user:  
```
$ adduser username
```
2. Grant sudoer rights:
```
$ usermod -aG sudo username
```
4. Update and upgrade (keeping local existing configuring files, if asked):
```
$ apt update && apt upgrade
```
5. Enable external access via ssh for the newly created user by copying the ssh public key:  
```
$ rsync --archive --chown=username:username ~/.ssh /home/username
```
6. Change ssh port from 22 to 2200:
```
$ nano /etc/ssh/sshd_config
```
Replace `#port 22` by `port 2200`.
7. Restart:
```
service ssh restart
```
8. ```exit```
9. Log in as the new via ssh:
```
ssh -i /path/to/private/sshkey username@ip.ad.re.ss -p 2200
```
10. Configure the firewall to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123) and OpenSSH:
```
$ sudo ufw default deny incoming  # deny all incoming connections
$ sudo ufw default allow outgoing # allow all outgoing connections
$ sudo ufw allow OpenSSH
$ sudo ufw allow 2200/tcp         # open port 2200 for ssh connection
$ sudo ufw allow www              # open port 80 for HTTP requests
$ sudo ufw allow 123/udp
$ sudo ufw deny 22                # close this port as it was replaced by 2200
$ sudo ufw enable
$ sudo ufw status                 # should list all the rules you just set up
```  
11. Disable root login  
```
$ sudo nano /etc/ssh/sshd_config
```  
Replace `PermitRootLogin yes` by `PermitRootLogin no`.
```
$ service ssh restart  
$ ssh -i /path/to/private/sshkey root@ip.ad.dr.ess -p 2200
```
Should not work anymore.
12. Configure Timezone to Use UTC  
```
sudo dpkg-reconfigure tzdata
```  
Select `none of the above`.

## 3. Install Apache web server  
```
$ sudo apt update  
$ sudo apt install apache2
```  
Check your website at http://ip.ad.dr.ess.
You should see the default apache 'it works' page.
## 4. Install mod_wsgi
```
$ sudo apt install libapache2-mod-wsgi
```    
## 5. Install pip
The application I am deploying is written in python 2.7.
```
$ sudo apt install python-pip  
```
Check with `pip --version` that the installation was successful.  
## 6. [Install and configure git](https://git-scm.com/download/linux)  
```
$ sudo add-apt-repository ppa:git-core/ppa
$ sudo apt update
$ sudo apt-get install git
$ git config --global user.name "username"
$ git config --global user.email "username@mail.com"  
```
You can check your settings with `git config -l`.
## 7. PostgreSQL
### 7.1 [Install](https://www.postgresql.org/download/linux/ubuntu/)   
```
$ sudo nano /etc/apt/sources.list.d/pgdg.list
```  
- Add `deb http://apt.postgresql.org/pub/repos/apt/ bionic-pgdg main`  
- Import the repository signing key  
```
$ wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
```
- Install
```
$ sudo apt-get update
$ sudo apt install postgresql-10
```
### 7.2 Configure:
- Create `catalog` role for the database:
```
$ sudo su - postgres
$ psql
postgres=# CREATE ROLE catalog WITH PASSWORD 'catalog';
postgres=# ALTER ROLE catalog CREATEDB;
```  
You can check with `\du` that the catalog user has been created.
- Quit and exit: `\q `, `exit `.
- Create catalog **user** and add to sudoers :
```
$ sudo adduser catalog
$ sudo usermod -aG sudo catalog
```
- Log as catalog and create catalog database:
```
$ su - catalog
$ createdb catalogue
```
You can check with `\l` that the database has been created.

## 8. Deploy application  
### 8.1 Set up the Flask app

1. Create application directory:
```
$ sudo mkdir /var/www/catalog && cd /var/ww/catalog/
```
2. Clone application repository, renaming it 'catalog':
```
$ sudo git clone https://github.com/Gry0u/vinyls_catalogue.git catalog
```
3. Rename and edit `views.py`
Change
```
app.debug = True
app.run(host='0.0.0.0', port=5000)
```
to
```
app.run()
```
4. Edit `database_setup.py`, change the engine creation statement to:  
`engine = create_engine('postgresql://catalog:catalog@localhost/catalog')`
5. Add your server's IP address to your  authorized origins in your OAuth client ID settings (I use the [Google API](https://console.developers.google.com/))
6. Update `client_secrets.json` in catalog/catalog accordingly (dowload file, then copy paste content). Check that the client_secret key
is in the json file.
7. Update login.html with your CLIENT ID if necessary
8. [Create a python virtual environment](http://flask.pocoo.org/docs/0.12/installation/ )
```
$ sudo apt install python-virtualenv
```
9. Install required modules  
In catalog/catalog/:
```
$ sudo virtualenv venv  
$ chown -R grader:grader catalog/
$ . venv/bin/activate
$ pip install --upgrade flask sqlalchemy httplib2 oauth2client requests  
$ sudo apt install libpq-dev  
$ pip install psycopg2
```

### 8.2 Set up and configure the virtualhost
1. Create `/etc/apache2/sites-available/catalog.conf` and add the following lines:
```
<VirtualHost *:80>
		ServerName your.server.ip.address
		ServerAdmin you@mail.com
		WSGIScriptAlias / /var/www/catalog/catalog.wsgi
		<Directory /var/www/catalog/catalog/>
			Order allow,deny
			Allow from all
		</Directory>
		Alias /static /var/www/catalog/catalog/static
		<Directory /var/www/catalog/catalog/static/>
			Order allow,deny
			Allow from all
		</Directory>
		ErrorLog ${APACHE_LOG_DIR}/error.log
		LogLevel warn
		CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
2. Enable the virtual host:
```
$ sudo a2ensite catalog
```
And reload as prompted
```
$ sudo systemctl reload apache2
```
### 8.3 Create the .wsgi file
```
$ sudo nano /var/www/catalog/catalog.wsgi
```
- Add the following lines:  
```
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/catalog/")

from catalog import app as application
application.secret_key = 'Add your secret key'
```
- Restart Apache: `sudo service apache2 restart`

$ sudo a2dissite 000-default.conf
$ sudo service apache2 restart









[intial server set up digital ocean ink](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-18-04?utm_source=local&utm_medium=Email_Internal&utm_campaign=Email_UbuntuDistroNginxWelcome&mkt_tok=eyJpIjoiTXpZNE5EaGpOR0ZoT1dFeiIsInQiOiIzc2ZXZVlHanE3Sk85ZWpmaEN3cUJSa21ZNlkwUGJJMVBZMkVkZHVmQTNZSzczUTdEYnpTYStTNWpaajg0MGJ3SUhHTlVSTlR3OGY5RzBhVlVBem5lRmp0WHJkdURHZmNuN0JHeTR3c2s0TjAxUlZcL043UDNXS3RIbndITTQwTFMifQ%3D%3D)

https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps
