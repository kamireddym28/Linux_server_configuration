# Linux Server Configuration -- FSDN Project6

> The objective of this project is to perform baseline installation of a Linux distribution on a virtual machine. Hosted a web applications, to include installing updates, securing it from a number of attack vectors and installing/configuring web and database servers.

## Info
- IP address: 54.187.202.55
- SSH Port : 2200
- Application URL : http://ec2-54-187-202-55.us-west-2.compute.amazonaws.com/

## Requirements

 - [**Vagrant**](https://www.vagrantup.com/)
 - [**VirtualBox**](https://www.virtualbox.org/wiki/Download_Old_Builds_5_1)
 - Amazon Lightsail instance to host our web application

## Implementation Procedure

**1. Login as a root user on VM**
- Visit Amazon lightsail account to obtain AWS IP address and key. Download ***Lightsailkey.pem*** and move it to ```~/.ssh``` folder on your local machine
- Open your terminal and type ```chmod 400 ~/.ssh/Lightsail-key.pem``` followed by ```ssh -i ~/.ssh/Lightsailkey.pem ubuntu@54.187.202.55```

**2. Add new user grader**
- ```sudo adduser grader```
- ```sudo nano /etc/sudoers.d/grader```
- Add following lines ```grader ALL=(ALL:ALL) ALL```

**3. Update existing installed packages**
- ```sudo apt-get update```
- ```sudo apt-get upgrade```

**4. Configuring the key based authentication for user grader**
- On your local machine generate public and private key and save it in the ```~/.ssh``` folder using ```ssh-keygen -f ~/.ssh/udacity-rsakey```
- On Virtual Machine:
    ```su - grader```
    ```mkdir .ssh```
    ```touch .ssh/authorized_keys```
    ```nano .ssh/authorized_keys```
- Copy the content of _udacity-rsakey.pub_ previously generated to this file.
- Provide permissions to the user grader:
    ```sudo chmod 700 .ssh```
    ```sudo chmod 644 .ssh/authorized_keys```
    ```sudo chown -R grader:grader .ssh```
- Reload SSH using ```sudo service ssh restart```
- At this point we should be able to log into the remote VM using ```ssh -i ~/.ssh/udacity-rsakey grader@54.187.202.55```

**5. Change SSH port from 22 to 2200**
- ```sudo nano /etc/ssh/sshd_config``` --> Modify port 22 to 2200 and save it.
- ```sudo service ssh restart```
- At this point we should be able to log into the remote VM using ```ssh -i ~/.ssh/udacity-rsakey grader@54.187.202.55 -p 2200```
**Add a custom application and tcp protocol with corresponding port set to 2200 in the networking section in your instance at amazon lightsail.

**6. Disabling ssh login for root use**
- ```sudo nano /etc/ssh/sshd_config``` --> Modify _PermitRootLogin_ to _no_ and save it
- ```sudo service ssh restart```

**7. Configuring Uncompatible Firewall (UFW)**
- Allow incoming connections for _SSH_ at port 2200, _HTTP_ at port 80 and _NTP_ at port 123.
- ```sudo ufw allow 2200/tcp```
- ```sudo ufw allow 80/tcp```
- ```sudo ufw allow 123/tcp```
- ```sudo ufw enable```
- ```sudo ufw status```

**8. Setting the local timezone to UTC**
- ```sudo dpkg-reconfigure tzdata``` and then choose UTC. 

**9.Install Apache and mod_wsgi**
- ```sudo apt-get install apache2```
- ```sudo apt-get install libapache2-mod-wsgi python-dev```
- ```sudo a2enmod wsgi``` to enable mod_wsgi
- ```sudo service apache2 start```

**10. Install git and cloning item_catalog from Github**
- ```sudo apt-get install git```
- ```cd /var/www```
- ```sudo mkdir catalog```
- ```sudo chown -R grader:grader catalog``` to provide created ```catalog``` ownership to grader
- cd /catalog
- ```git clone https://github.com/kamireddym28/Item_Catalog_Project.git catalog```
- ```sudo nano catalog.wsgi``` and then paste the following in _catalog.wsgi_:
    ```
    import sys
    import logging
    logging.basicConfig(stream=sys.stderr)
    sys.path.insert(0, "/var/www/catalog/")
    
    from catalog import app as application
    application.secret_key = 'supersecretkey'
    ```
- Rename Catalog_project.py to ```__init__.py``` using ```sudo mv application.py __init__.py```.

**11. Install Virtual Environment**
- ```sudo pip install virtualenv```
- ```sudo virtualenv venv```
-  ```source venv/bin/activate```
-  ```sudo chmod -R 777 venv```

**12. Install Flask and supporting dependencies**
- ```sudo apt-get install python-pip```
- ```pip install Flask```
- ```sudo pip install httplib2 oauth2client sqlalchemy psycopg2 sqlalchemy_utils```

**13. Updating _redirect-uris_ in _client_secrets.json_ and path in ```__init__.py```**
- ```sudo nano __init__.py```
- Change **client_secrets.json** path to ```/var/www/catalog/catalog/client_secrets.json```.
- Change **fb_client_secrets.json** path to ```/var/www/catalog/catalog/fb_client_secrets.json```.
- Update the _redirect-uris_ and _javascript-origins_ in _client_secrets.json_ with this new URL ```http://ec2-54-187-202-55.us-west-2.compute.amazonaws.com/```.

**14. Enable new Virtual host**
- ```sudo nano /etc/apache2/sites-available/catalog.conf```
- Paste the following:
    ```
    <VirtualHost *:80>
        ServerName 54.187.202.55
        ServerAlias ec2-54-187-202-55.us-west-2.compute.amazonaws.com
        ServerAdmin admin@54.187.202.55
        WSGIDaemonProcess catalog python-path=/var/www/catalog:/var/www/catalog/venv/lib/python2.7/site-packages
        WSGIProcessGroup catalog
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
- ```sudo a2ensite catalog``` to enable virtual host.

**15. Install and Configure PostgreSQL***
- ```sudo apt-get install libpq-dev python-dev```
- ```sudo apt-get install postgresql postgresql-contrib```
- ```sudo su - postgres```
- ```psql```
- ```CREATE USER catalog WITH PASSWORD 'mypassword';```
- ```ALTER USER catalog CREATEDB;```
- ```CREATE DATABASE catalog WITH OWNER catalog;```
- ```\c catalog```
- ```REVOKE ALL ON SCHEMA public FROM public;```
- ```GRANT ALL ON SCHEMA public TO catalog;```
- ```\q```
- ```exit```
- Update existing engine path to ```engine = create_engine('postgresql://catalog:mypassword@localhost/catalog')``` in _database_setup.py_ , _modelcatalog.py_ and __init__.py.
- ```sudo python database_setup.py```
- ```sudo python modelcatalog.py```
- ```sudo service apache2 restart```
- ```sudo python __init__.py``` then visit http://54.187.202.55

## References
- Udacity course on **Configure Web Servers**.
- http://swaroopsm.github.io/12-02-2012-Deploying-Python-Flask-on-Apache-using-mod_wsgi.html
- https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps
- Udacity discussion forum.

