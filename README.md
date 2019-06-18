# Ud-Linux-Server

## Description

Goals of the project given by our instructors from Udacity:
This project takes a baseline installation of a Linux distribution on a virtual machine and prepares it to host a web application (https://github.com/trekirkman/Udacity-Fullstack-Catalog-Project). This includes securing it from a number of attack vectors and installing/configuring web and database servers.

## Server Info

Public IP address: 34.220.197.8.

URL: [34.220.197.8.nip.io](http://34.220.197.8.nip.io) (nip.io necessary to authenticate via Google Oauth)

Accessible SSH port: 2200.

## Configuration Walkthrough

### 1. Secure Server
- Update all currently installed packages

        sudo apt-get update
        sudo apt-get upgrade
 
- Change the SSH port from 22 to 2200. Make sure to update Lightsail firewall settings accordingly

        sudo nano /etc/ssh/sshd_config
        # Change Port 22 to 2200

- Configure Firewall (UFW) Settings
Project requirements need the server to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123).

        sudo ufw allow 2200/tcp
        sudo ufw allow 80/tcp
        sudo ufw allow 123/udp
        sudo ufw enable

### 2. Create user named *grader* and grant this user sudo permissions.

- Add a new user called *grader*:

        sudo adduser grader

- Create a new file under the sudoers directory: 

        sudo nano /etc/sudoers.d/grader
        
        # Add the following text:
        grader ALL=(ALL:ALL) ALL

### 3. Configure key-based authentication
- On your local machine, generate an encryption key with the ssh-keygen tool

        ssh-keygen
        
- Create an authorized keys file and add the newly generated public key:

        touch /home/grader/.ssh/authorized_keys
        # Paste content from public key into this file

- Update permissions:
        
        sudo chmod 700 /home/grader/.ssh
        sudo chmod 644 /home/grader/.ssh/authorized_keys
        sudo chown -R grader:grader /home/grader/.ssh
 
 Now you  can login with: 
 
        ssh grader@34.220.197.8 -p 2200 -i ~/ssh/linuxCourse

- Enforce key-based authentication

        sudo nano /etc/ssh/sshd_config
        # Set PasswordAuthentication to *no*
        sudo service ssh restart

- Disable login for root user

        sudo nano /etc/ssh/sshd_config
        # Change PermitRootLogin to no
        sudo service ssh restart

### 4. Configure the local timezone to UTC

Open time configuration GUI and set it to UTC with: `$ sudo dpkg-reconfigure tzdata`.

### 5. Install Apache & Mod_wsgi

        # Install Apache
        sudo apt-get install apache2
        # Install mod_wsgi
        sudo apt-get install libapache2-mod-wsgi-py3
        # Enable mod_wsgi
        sudo a2enmod wsgi
        # Start apache
        sudo service apache2 start


### 6. Clone the Web Application from Github
- Install & Configure Git

        sudo apt-get install git
        # Configure username & email
        git config --global user.name <username>
        git config --global user.email <email>

- Clone the Catalog App

        cd /var/www
        sudo mkdir catalog
        # Change owner
        sudo chown -R grader:grader catalog
        cd /catalog
        # Clone catalog repo from Github
        git clone https://github.com/trekirkman/Udacity-Fullstack-Catalog-Project.git

- Create a .wsgi file to serve the application and add the following:

        import sys
        import logging
        logging.basicConfig(stream=sys.stderr)
        sys.path.insert(0, "/var/www/catalog/")
        
        from catalog import app as application

### 7. Install Flask and the other project dependencies
- Install pip, a tool for managing Python packages
        
        sudo apt-get install python-pip
        
- Install Dependencies
        
        # Flask
        pip install Flask
        # Other Packages
        pip install httplib2 request oauth2client sqlalchemy python-psycopg2

### 8. Configure and enable a new virtual host

- Navigate to the config file
        
        sudo nano /etc/apache2/sites-available/catalog.conf

- Paste the code below within the Virtual Host Block
```
<VirtualHost *:80>
    ServerName 34.220.197.8
    WSGIDaemonProcess catalog
    WSGIProcessGroup catalog
    WSGIScriptAlias / /var/www/catalog/catalog.wsgi
    <Directory /var/www/catalog/catalog/>
        Order allow,deny
        Allow from all
        Option -Indexes
    </Directory>
</VirtualHost>
```

### 9. Install and configure PostgreSQL

- Install PostgreSQL

        sudo apt-get install postgresql
        
- Limit remote connections to the database

        sudo nano /etc/postgresql/9.5/main/pg_hba.conf
        
- Create new user named catalog

        # Connect to database
        psql
        # Create new user with DB permissions
        CREATE USER catalog WITH PASSWORD 'udacity';
        ALTER USER catalog CREATEDB;
        CREATE DATABASE catalog WITH OWNER catalog
        # Connect to the catalog DB
        \c catalog
        # Lock down rights & permissions
        REVOKE ALL ON SCHEMA public FROM public;
        GRANT ALL ON SCHEMA public TO catalog;
        
 Note: database connections are now performed with the following code (in python):
 
        engine = create_engine('postgresql://catalog:udacity@localhost/catalog')
 
- Populate the database

        python3 sample_data.py


### 10. Restart Apache and Launch

        sudo service apache2 restart
 
 URL: [34.220.197.8.nip.io](http://34.220.197.8.nip.io)
 
 ## Resources
 - https://lightsail.aws.amazon.com/ls/docs/en_us/articles/getting-started-with-amazon-lightsail
 - http://flask.pocoo.org/docs/0.12/deploying/mod_wsgi/
 - http://httpd.apache.org/docs/current/configuring.html
 - https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps
 - https://stackoverflow.com/questions/36020374/google-permission-denied-to-generate-login-hint-for-target-domain-not-on-localh
 - https://stackoverflow.com/questions/31168606/internal-server-error-target-wsgi-script-cannot-be-loaded-as-python-module-and
 - https://knowledge.udacity.com/questions/8014
