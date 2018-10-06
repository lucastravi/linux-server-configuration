# LINUX SERVER CONFIGURATION PROJECT
### Project 5 - Full Stack Web Developer Nanodegree Udacity
The aim of this project is to deploy a Python-Flask web applicaton (Item Catalog - developed in Project 3) on the Amazon Lightsail platform following specific security settings.
<hr>

## Acessing the server
Public IP address: http://18.213.113.249

## 1 - Creating the istance and obtaining the first key
1. Login in the Amazon Lightsail Platform;
2. Create a new instance;
3. Select your location;
4. Select a platform (Linux/Unix);
5. Select a blueprint (Ubuntu 16.04 LTS);
6. Download the default ssh key provided by Amazon;
7. Chose the pricing plan;
8. Give the instance a name (item-catalog)
9. Go to the folder where the default key has been downloaded:
```
mv ~/LightsailDefaultPrivateKey-us-east-1.pem ~/.ssh/LightsailDefault.pem
chmod 700 ~/.ssh/
chmod 600 ~/.ssh/LightsailDefault.pem
```
10. Then log into the server as the default user (ubuntu):
```
ssh ubuntu@18.213.113.249 -i ~/.ssh/LightsailDefault.pem
```

## 2 - Creating grader with sudo access
1. Update and upgrade the packages:
```
sudo apt-get update 
sudo apt-get upgrade
```
2. Create the user grader:
```
sudo adduser grader
```
3. Give grader sudo access:
```
sudo visudo
```
*In the section of user privilege user specification add:*
```
grader ALL=(ALL:ALL) ALL
```
4. Now login as grader with sudo access:
```
sudo login grader
```

## 3 - Configuring the ports and getting the SSH key pair:
1. On the Amazon Lightsail platform, go to the Networking properties of your instance and create a new custom Port with protocol TCP and Port Range 2200;
2. Go to the sshd config file and change the ports, besides that, also remove the root remote login:
```
# What ports, IPs and protocols we listen for
# Port 22
Port 2200

# Authentication:
LoginGraceTime 120
#PermitRootLogin prohibit-password
PermitRootLogin no
StrictModes yes

# Change to no to disable tunnelled clear text passwords
PasswordAuthentication No
```
3. Create the ssk key pair on your local machine:
```
$ ssh-keygen
```
4. Use the standard folder and name the key pair as catalog_rsa. Two files will be generated inside a .ssh folder:
- catalog_rsa
- catalog_rsa.pub
5. Back to the server, go to the grader folder ```cd /home/grader/``` create an authorized_keys file to receive the public key content:
```
mkdir .ssh
cd .ssh
sudo nano athorized_keys
```
6. On your local machine:
```
$ nano ~/.ssh/catalog_rsa.pub
```
Copy the content of this file and paste inside the authorized_keys file in the server
7. Now is possible to access the server using the private key from the ssh key pair from your local machine:
```
$ ssh grader@18.213.113.249 -p 2200 -i ~/.ssh/catalog_rsa
```

## 4 - Configuring the firewall and the timezone:
1. Check if UFW is active:
```
sudo ufw status
```
2. Add the firewall rules:
```
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2200/tcp
sudo ufw allow 80/tcp or sudo ufw allow www (either one of these commands will work)
sudo ufw allow 123/udp
```
3. Enable the rules:
```
sudo ufw enable
```
4. Configure the timezone:
```
sudo dpkg-reconfigure tzdata
```
Chose None of the above and select the your UTC timezone

## 5 - Clone the application and install the required packages:
1. Install all the required packages:
```
sudo apt-get install git
sudo apt-get install python-pip python-flask python-sqlalchemy python-psycopg2
sudo pip install oauth2client requests httplib2
```
2. Create a directory to receive the application:
```
cd /var/www/
sudo mkdir catalog
cd catalog
```
3. Clone the application from Github and rename the folder:
```
sudo git clone https://github.com/lucastravi/item-catalog-udacity.git
sudo mv item-catalog-udacity catalog
```
4. Rename the app.py file:
```
sudo mv app.py __init__.py
```

## 6 - Install and configure Apache
1. Install the apache package:
```
sudo apt-get install apache2
sudo apt-get install libapache2-mod-wsgi python-dev
```
2. Enable mod_wgsi:
```
sudo a2enmod wsgi 
```
3. Create and configure the item catalog virtual host:
```
sudo nano /etc/apache2/sites-available/catalog.conf
```
And add the following code:
```
<VirtualHost *:80>
		ServerName 18.213.113.249
		ServerAdmin travi.lucas@gmail.com
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
4. Create and configure the .wgsi file:
```
cd /var/www/catalog
sudo nano catalog.wsgi
```
And add the following cod:
```
import sys
import logging

logging.basicConfig(stream=sys.stderr)
sys.path.append('/var/www/catalog')
sys.path.append('/var/www/catalog/catalog')

from catalog import app as application
application.secret_key = 'DEV_SECRET_KEY'
```
5. Disable the default site and enable the catalog application:
```
sudo a2dissite 000-default
sudo a2ensite catalog
sudo service apache2 reload
```
## 7 - Install and configure PostgreSQL and finalize the lauching:
1. Install PostgreSQL and set a password:
```
sudo apt-get install postgresql
sudo passwd postgres
```
2. Login with postgres user:
```
su postgres
psql
```
3. Create and configure the application database and quit:
```
>> CREATE USER catalog;
>> ALTER USER catalog WITH PASSWORD 'catalog';
>> CREATE DATABASE catalog WITH OWNER catalog;
>> \c catalog
>> REVOKE ALL ON SCHEMA public FROM public;
>> GRANT ALL ON SCHEMA public TO catalog;
>> \q
```
4. Log with grader again and restart PostgreSQL:
```
su grader
sudo service postgresql restart
```
5. For all the files with ```engine = create_engine('sqlite:///itemcatalog.db')``` switch this line of the code for ```engine = create_engine('sqlite:///itemcatalog.db')```
6. In the __init__.py file the, update the path to the client_secrets.json file:
```
CLIENT_ID = json.loads(
    open('/var/www/itemsCatalog/vagrant/catalog/client_secrets.json', 'r').read())['web']['client_id']

oauth_flow = flow_from_clientsecrets('/var/www/itemsCatalog/vagrant/catalog/client_secrets.json', scope='')
```
7. Launch the application:
```
sudo service apache2 restart
```
If something go wrong, debug is on ```/var/log/apache2/error.log```.
<hr>
## References:
1. https://medium.com/@mariasurmenok/creating-a-server-with-amazon-lightsail-11c377cf814c
2. https://github.com/harushimo/linux-server-configuration
3. https://github.com/SteveWooding/fullstack-nanodegree-linux-server-config
4. https://github.com/AbigailMathews/FSND-P5
6. https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps
7. https://lightsail.aws.amazon.com/ls/docs/en/all
