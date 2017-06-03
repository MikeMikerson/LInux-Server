# Server info
* Public IP Address: 52.32.240.84
* URL: http://ec2-52-32-240-84.us-west-2.compute.amazonaws.com/

# Guide to how I set things up
## First, create a new user, "grader"

`sudo adduser grader`

## Grant grade sudo access

As the user Ubuntu, create a new grader file with `sudo nano /etc/sudoers.d/grader`

Within the editor add the following line:

`grader ALL=(ALL) NOPASSWD:ALL`

Next, log into the grader user account.

## Update packages
Update all packages with the following commands:
```
sudo apt-get update
sudo apt-get upgrade
```

## Locally generate key pair
To generate a key pair locally do `ssh-keygen` and store it into `/home/USERNAME/.ssh/FILENAME`

## Put key pair on server
As the grader user, create a directory to put the key into with `mkdir .ssh`, then create a file to put the key into with `touch .ssh/authorized_keys`. Next copy the contents of the generated keygen within (local) `/home/USERNAME/.ssh/FILENAME.pub` into authorized_keys.

Last and definitely not least, secure the directory and key.
```
chmod 700 .ssh
chmod 644 .ssh/authorized_keys
```

## Disable password logins
Open the editor with `sudo nano /etc/ssh/sshd_config`.

If "PasswordAuthentication" is set to "Yes", simply change it to "no". Exit the editor and save. Don't forget to restart the service with `sudo service ssh restart`.

## SSH Configuration
Open up `/etc/ssh/sshd_config` with `sudo nano /etc/ssh/sshd_config`.

We'll be using port 2200 instead of 22, so change the port:

`Port 2200`

Make sure PasswordAuthentication is set to no.

`PasswordAuthentication no`

Disable root login.

`PermitRootLogin no`

Next `sudo service ssh restart` so it takes effect.

To test if everything works, you can attempt to log from your local terminal with ssh grader@35.164.105.239 -p 22 PATH_TO_KEY.PUB (the locally generated key from before). If everything goes well, you should login to the grader user.

## Firewall setup
First, make sure the default is to deny incoming request.

`sudo ufw default deny incoming`

Next, allow outgoing calls.

`sudo ufw default allow outgoing`

First, make sure you allow ssh connections. Without this, chances are you won't be able to log in.

`sudo ufw allow 2200/tcp`

Allow http requests.

`sudo ufw allow www`

Allow NTP requests

`sudo ufw allow ntp`

Enable ufw

`sudo ufw enable`

In Amazon Lightsail, be sure to make the same changes as above. This can be done by clicking "Add another" and putting in custom ports.

# Configure local timezone to UTC
To configure local timezone:

```
sudo dpkg-reconfigure tzdata
```

Select "None of the above" and then "UTC".

# Apache and postgresql setup

Install apache and postgresql with the following commands:

Check for remote connections in `/etc/postgresql/9.5/main/pg_hba.conf`.

# Setup Item Catalog
After installing postgreql, create a user and a database to be used for the item catalog.

```
sudo -u postgres createuser -P catalog
sudo -u postgres createdb -O catalog catalog
```

After the first command, enter a password to be used later.

## Required installations
Install the following python packages:
```
sudo apt-get install python-psycopg2
sudo apt-get install python-flask
sudo apt-get install python-pip
sudo apt-get install python-sqlalchemy
```

Next, install the following with pip:

```
sudo pip install --upgrade setuptools
sudo pip install bleach
sudo pip install oauth2client
sudo pip install requests
sudo pip install httplib2
sudo pip install redis
sudo pip install passlib
sudo pip install itsdangerous
sudo pip install flask-httpauth
sudo pip install flask-seasurf
```

Last but not least, install git

```
sudo apt-get install git
```

## Clone catalog repository
First, cd into the /srv directory. Everything in this directory will need to be run with sudo.
```
cd /srv
```
Next, clone the catalog repository:
```
sudo git clone https://github.com/MikeMikerson/Server-Item-Catalog.git
```

# Changes to item catalog
* Change main.py to main.wsgi

## New virtual host
Create a conf file like so:
```
sudo nano /etc/apache2/sites-available/CatalogApp.conf
```

Put the following into the file:

```
<VirtualHost *:80>
	#ServerName www.something.com

	ServerAdmin webmaster@localhost

	WSGIScriptAlias / /srv/catalog-app/Server-Item-Catalog/main.wsgi
	<Directory /srv/catalog-app/Server-Item-Catalog/>
		Require all granted
	</Directory>
	Alias /static /var/www/FlaskApp/FlaskApp/static
	<Directory /var/www/FlaskApp/FlaskApp/static/>
		Require all granted
	</Directory>
	ErrorLog ${APACHE_LOG_DIR}/error.log
	LogLevel warn
	CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

After that, disable the default host and enable the CatalogApp:

```
sudo a2dissite 000-default.conf
sudo a2ensite CatalogApp.conf
sudo service apache2 reload
```

## Setup database
Within /project/db do `python database_setup.py` to set the database up.

## Change path for oauth
There's a need to change the path of the client secrets.

For Google client secrets:

```
/srv/Server-Item-Catalog/client_secrets.json
```

and for FB client secrets:

```
/srv/Server-Item-Catalog/fb_client_secrets.json
```