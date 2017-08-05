
# Linux Server Configuration

Prepare the server to host your web applications.

#### IP Address

ipaddress:13.126.118.210 
SSH Port: 2200

#### Publicly available URL

[http://ec2-13-126-118-210.ap-south-1.compute.amazonaws.com](http://ec2-52-66-164-206.ap-south-1.compute.amazonaws.com)

## Key Points

- Deploying your web applications to a publicly accessible server is the first step in getting users

- Properly securing your application ensures your application remains stable and that your user’s data is safe

## Project Details

* Updating all packages on the server
	- sudo apt-get update 
	- sudo apt-get upgrade

* Update timezone
	

* Create a new user account named *"grader"*
	- sudo adduser grader
	- Install finger (if you want to)
		

* Give grader access
	- give grader the permission to sudo
	- *usermod -aG grader sudo*

* Secure your server
	- Change the SSH port from 22 to 2200.
		- configure sshd_config file to change the port
		- sudo nano /etc/ssh/sshd_config
		- Also, change PermitRootLogin from prohibited-password to no
		- sudo service ssh restart

	- Configure *Firewall*
		- Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)
			```sh
			- sudo ufw status
			- sudo ufw default deny incoming
			- sudo ufw default allow outgoing
			- sudo ufw allow 2200/tcp
			- sudo ufw allow 80/tcp
			- sudo ufw allow 123/udp
			- sudo ufw enable
			```

* When you change the SSH port, the Lightsail instance will no longer be accessible through the web app 'Connect using SSH' button. The button assumes the default port is being used.
Instructions for connecting from your terminal to the instance.
	- Download the key-pair for your instance from lightsail.
	- Then grant it the specific permissions
		- *chmod 600 key-pair.pem*
	- Also, to sort out the initial login issue
		- sudo nano /etc/ssh/sshd_config
		- set Password Authentication to "yes" (which we'll change later)
		- sudo service ssh restart
	- ssh grader@13.126.118.210 -p 2200 -i *key-pair.pem*

* Give grader access
	- Give grader the permission to sudo
		- sudo usermod -aG sudo grader

	- Create an SSH key pair for grader using the ssh-keygen tool.
		- On your local machine, generate SSH key pair with: ssh-keygen
		- Save `*yourkeygen*` file in your ssh directory /c/Users/<usernam>/.ssh/ example full file path that could be used: /home/Users/<username>/.ssh/item-catalog
		- You can add a password/passphrase to use in case your keygen file gets compromised(you will be prompted on your terminal to enter this password when you connect with key-pair)
		- login into grader account 
			- *ssh grader@13.126.118.210 -p 2200 -i *key-pair.pem*
			- Make .ssh directory
				- mkdir .ssh
				- make file to store key: *touch .ssh/authorized_keys*
		- on your local machine, read contents of the public key `cat .ssh/keygen.pub`
		- copy the key and paste in the file you just created in *grader* nano .ssh/authorized_keys
		- Save file
		- Set permissions of the .ssh directory and the authorized_keys file
				```sh
				- sudo chmod 700 .ssh
				- sudo chmod 644 .ssh/authorized_keys
				```
				- Set Password Authentication to "no" (which we had changed earlier to yes)

	- Login with key-pair
		- `ssh grader@Public-IP-Address* -p 2200 -i ~/.ssh/keygen`

* Prepare to deploy your project
	- Install git.
		- *sudo apt-get install git*

	- Install and configure Apache to serve a Python mod_wsgi application.
		- sudo apt-get install apache2
		- install mod_wsgi
			- sudo apt-get install libapache2-mod-wsgi
		- Configure Apache to handle requests using the WSGI module.
			- sudo nano /etc/apache2/sites-enabled/000-default.conf
		- add `WSGIScriptAlias / /var/www/html/myapp.wsgi` before *</VirtualHost>* closing line
		- Restart Apache
			- sudo apache2ctl restart
		- Install python dev and verify WSGI is enabled
			- Install python-dev package: *sudo apt-get install python-dev*
			- Verify wsgi is enabled: *sudo a2enmod wsgi*

	- Install and configure PostgreSQL:
		- Do not allow remote connections
		- Create a new database user named catalog that has limited permissions to your catalog application database.

* Deploy the Item Catalog project.
	- Clone and setup your Item Catalog project from the Github repository
	- Set it up in your server so that it functions correctly when visiting your server’s IP address in a browser. 
	- Make sure that your .git directory is not publicly accessible via a browser

	- Now, Clone Item Catalog project
		- cd /var/www
		- sudo mkdir catalog
		- cd catalog
		- git clone https://github.com/<your_project_path>/
		- sudo *touch catalog.wsgi*
			- Paste the following in catalog.wsgi file:
			```js
				#!/usr/bin/python
				import sys
				import logging
				logging.basicConfig(stream=sys.stderr)
				sys.path.insert(0,"/var/www/catalog/")

				from catalog import app as application
				application.secret_key = 'Add your secret key'
			```

	- Install all the dependencies for Item Catalog project
		```sh
		- sudo apt-get -qqy install postgresql python-psycopg2
		- sudo apt-get install python-pip
		- sudo pip install requests
		- sudo pip install oauth2client
		- sudo pip install htpplib2
		- sudo apt-get -qqy install python-sqlalchemy
		- pip install werkzeug==0.8.3
		- pip install flask==0.9
		- pip install Flask-Login==0.1.3
		```

	- Configure & Enable New Virtual Host
		- Create host config file
			- *sudo nano /etc/apache2/sites-available/catalog.conf*
			- And paste:
				- ```js
					<VirtualHost *:80>
					  ServerName 13.126.118.210
					  ServerAdmin admin@13.126.118.210
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
			- save the file
		- Enable *sudo a2ensite catalog*

	- `sudo service apache2 restart`

	- Install and setup PostgreSQL
		- sudo apt-get install postgresql
		- sudo -i -u postgres
		- psql
		- Create user catalo
			- *CREATE USER catalog WITH PASSWORD 'catalog';*
		- Change role of user catalog to createDB
			- *ALTER USER catalog CREATEDB;*
		- Create new DB `catalog` with own of catalog
			- *CREATE DATABASE catalog WITH OWNER catalog;*
		- Connect to database
			- *\c catalog*
		- Revoke all rights
			- *REVOKE ALL ON SCHEMA public FROM public;*
		- Give accessto only catalog role
			- *GRANT ALL ON SCHEMA public TO catalog;*
		- exit

	- Edit and config database_setup.py
		```js
		- sudo nano database_setup.py
		- engine = create_engine('postgresql://catalog:catalog@localhost/catalog')
		```

	- Edit and config __init__.py
		- sudo nano __init__.py
		- engine = create_engine('postgresql://catalog:catalog@localhost/catalog')

* Also, add the public IP address back into the Google Developer console and Facebook developer console
