#Linux Server Configuration

This is udacity Fifth project, we will configure an Ubuntu instance from amazon lightsail, and install relevant packages to run our Item's catalog app we created last project.
 
 IP: 3.121.241.157
 PORT: 2200
 URL: http://3.121.241.157.xip.io/

## install Packages

first update the repo and install the latest version of installed binaries.
`sudo apt update`
`sudo apt upgrade`
then install relevent packges.
`sudo apt install postgresql`, because we will use postgres database.
`sudo apt install apache2 libapache2-mod-wsgi-py`, because we will serve our application using apache and mod_wsgi
`sudo apt install python python-pip`, python 2.7 and pip

## Firewall
`sudo ufw allow 2200/tcp` for ssh
`sudo ufw allow 123/tcp` for ntp
`sudo ufw allow 80/tcp` for http
`sudo ufw default deny incoming` to deny others port
` sudo ufw enable` enable the firewall

## add user grader

`sudo useradd grader -m`,  create a user and create a home directory
`cd /home/grader`, change directory 
`sudo su - grader` to change into user grader.
`mkdir .ssh && touch .ssh/authorized_keys`, create .ssh directory and create empty authorized_keys that will be used later to hold our public key 
`ssh-keygen` follow the instruction to create a ssh keys.
`cat /home/grader/.ssh/id_rsa.pub >>  /home/grader/.ssh/authorized_keys` put public key content to authenticate using the new private key.
now copy `id_rsa` to your local machine for authentication, i copy and paste it in my local environments and placed the correct permission using chmod : `chmod 600 copied_id_rsa`
`echo "grader	ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers` give grader sudo privilege on all bin's without password.

## ssh

edit /etc/ssh/sshd_config.
change the configuration to:
From `#Port 22` To `Port 2200`
`PasswordAuthentication no`, you don't need to change pass authentication to no because the instance already configured to not accept clear password.
From `#PermitRootLogin prohibit-password` to `PermitRootLogin no`
to deny root access from ssh.

`service ssh restart`

## Amazon Lightsail
* visit your instance in https://lightsail.aws.amazon.com.
* in Networking tab change the Firewall configuration:
Delete all rules, and add HTTP:80/TCP, Custom:123/TCP, Custom:2200/TCP

![](http://i.imgur.com/yQyOwsx.png====) 

## postgresql
* `service postgresql start` to start postgresql.
* ` sudo -u postgres createuser fsnd --pwpromp`, create a DB user with username fsnd, and will be prompted for a password.
* `sudo -u postgres createdb catalog`, to  create the DB
* ` sudo -u postgres psql`  & and enter `grant all privileges on database catalog to fsnd ;`, will grant privilege to our newly created user to control catalog db.


## Google OAuth
* go to https://console.developers.google.com
* add the domain `xip.io` into the authorized domians in Credentials -> OAuth consent screen
* edit Client ID in Credentials -> Credentials and add `http://3.121.115.151.xip.io`  in both Authorized Javascript Origins and Authorized redirect URLs.
* Download Json config file

## Web APP
* `cd/var/www/html`
* `git clone https://github.com/0xSensei/FSND-itemcatalog.git FSND && cd FSND`, clone the app.
* `pip install virtualenv && virtualenv venv && source venv/bin/activate`, install virtualenv and initialize the environment and active it.
* `pip install -r requirements.txt`, install the dependency.
*  edit the connection string in `project.py` & `database_setup.py` & `database_setup.py` for postgresql. Example: 
` create_engine('postgresql://username:password@HOST/DB_NAME')`
* `pip install psycopg2` postgresql driver.
*  place newly edited google json oauth credentials into the project.
* ` mv peoject.py __init__.py` rename the project file for easier import later in wsgi config.

## apache mod_wsgi

* create a new site conf `touch /etc/apache2/sites-available/items.conf`
 and write this config:

 >
	 <VirtualHost *:80>
		ServerName 3.121.115.151.xip.io
		ServerAdmin admin@juaythin.com
		WSGIScriptAlias / /var/www/html/FSND/conf.wsgi
		<Directory /var/www/html/FSND/>
			Order allow,deny
			Allow from all
			Require all granted
		</Directory>
		Alias /static /var/www/html/FSND/static
		<Directory /var/www/html/FSND/static/>
			Order allow,deny
			Allow from all
		</Directory>
		ErrorLog ${APACHE_LOG_DIR}/error.log
		LogLevel warn
		CustomLog ${APACHE_LOG_DIR}/access.log combined
	</VirtualHost>


here basically we define where our wsgi script is, and placed the correct ACL in the target directory, and defined a symlink `/static` to our static dir for serving static files.
* create a new file in our project directory:
`touch /var/www/html/FSND/conf.wsgi` and write these lines:

>
	#!/usr/bin/python
	import sys
	import logging
	logging.basicConfig(stream=sys.stderr)
	sys.path.insert(0,"/var/www/html/")
	activate_this = '/var/www/html/FSND/venv/bin/activate_this.py'
	execfile(activate_this, dict(__file__=activate_this))
	from FSND import app as application
	application.secret_key = 	'laksdlkajsdlkjasdlkjalksdnlkansdlnasdkanlsdknaslkdn'
	
first we set up our path to that we cat import out app from FSND directory, then we activate our virtual environment that have our dependency installed and mod_wsgi will serve our app.
* sudo a2ensite items && service apache2 restart` to enable our site and restart apache to take effect. 

## references
* https://www.digitalocean.com/community/tutorials/how-to-setup-a-firewall-with-ufw-on-an-ubuntu-and-debian-cloud-server
* https://medium.com/coding-blocks/creating-user-database-and-adding-access-on-postgresql-8bfcd2f4a91e
* https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys-on-ubuntu-1804
