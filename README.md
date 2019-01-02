*This README displays the steps on how to deploy a web application in a Linux server on AWS.*

Public IP: 34.219.250.55

Hostname:
http://ec2-34-219-250-55.us-west-2.compute.amazonaws.com

Please use the hostname to access the catalog web application

# 1. Get your server
- Create an account in AWS Lightsail on their website.

- Version: Linux ubuntu 16.04

- To log in, you can use the 'Connect using SSH' button in AWS instance page or from the local Bash terminal

- Download the private key from the AWS SSH Key Pairs section into the local folder **[HOME_LOCAL_TERMINAL]/.ssh**

  The file is with the extension .pem

- **To log in**,`$ ssh ubuntu@[PUBLIC_IP] -p 22 -i ~/.ssh/[PRIVATE_KEY.pem]`

# 2. Secure your server
Once logged in with *ubuntu* user,

- `$ sudo apt-get update`

- `$ sudo apt-get upgrade`

 To change the SSH port from **22** to **2200**, first handle the firewall settings:
```bash
$ sudo ufw default deny incoming
$ sudo ufw default allow outgoing
$ sudo ufw allow ssh
$ sudo ufw allow 2200/tcp
$ sudo ufw allow www
$ sudo ufw allow 123/udp
$ sudo ufw deny 22
$ sudo ufw enable
```
```bash
$ sudo ufw status
To                         Action      From
--                         ------      ----
22                         DENY        Anywhere
2200/tcp                   ALLOW       Anywhere
80/tcp                     ALLOW       Anywhere
123/udp                    ALLOW       Anywhere
22 (v6)                    DENY        Anywhere (v6)
2200/tcp (v6)              ALLOW       Anywhere (v6)
80/tcp (v6)                ALLOW       Anywhere (v6)
123/udp (v6)               ALLOW       Anywhere (v6)
```

- Change manually from port from **22** to **2200** in `$ sudo nano /etc/ssh/sshd_config`


- Add these custom connections in AWS web instance:
```bash
Custom UDP 123
Custom TCP 2200
```

- Delete the ```SSH TCP 22``` element in the AWS web instance

- Reboot the server

- SSH connection is done via **2200** from now on:
`
$ ssh ubuntu@[PUBLIC_IP] -p 2200 -i ~/.ssh/[PRIVATE_KEY.pem]`

# 3. Give grader access

- To create username 'grader' `$ sudo adduser grader`

- To give grader the permission to sudo, `$ sudo nano /etc/sudoers.d/grader`and add this line: ```grader ALL=(ALL:ALL) ALL```

- In `$ sudo nano /etc/ssh/sshd_config`, **PermitRootLogin no** should be there - root should not be used for remote log-in


- `$ sudo apt-get update`
- `$ sudo apt-get upgrade`


- To create an SSH key pair for 'grader': In the local Bash terminal, enter `$ ssh-keygen`
- Generate and store the public and private keys in **~/..ssh**

- In the terminal of 'ubuntu', switch to username 'grader' by `$ su - grader`

```bash
$ mkdir .ssh
$ touch .ssh/authorized_keys
$ sudo nano .ssh/authorized_keys
```
- Copy paste the public key content into **.ssh/authorized_keys** and save.

- Reboot Linux via AWS web instance

- I can now log into Linux with 'grader' via
`$ ssh grader@[PUBLIC_IP] -p 2200 -i ~/.ssh/[PRIVATE_KEY]`

- After logging in with 'grader', change `$ sudo nano /etc/ssh/sshd_config` **PasswordAuthentication -> no**

- And change these file permissions:
```bash
$ chmod 700 .ssh
$ chmod 664 .ssh/authorized_keys
```
# Prepare to deploy your project
Please log into Linux with the username *'grader'* and continue with the next steps.

- Select UTC if not already set as UTC in `$ sudo dpkg-reconfigure tzdata`

- Install Apache server
`$ sudo apt-get install apache2`

  Test `http://34.219.250.55` in the browser - default page should be displayed

- Install wsgi,
```bash
$ sudo apt-get install libapache2-mod-wsgi
$ sudo a2enmod wsgi
```

- Install and configure PostgreSQL, `$ sudo apt-get install postgresql`

- Check if no remote connections are allowed `$ sudo cat /etc/postgresql/9.5/main/pg_hba.conf`

- `$ sudo su - postgres`   # log in with the super user
- `$ psql`     # connect to database

```
postgres=# CREATE DATABASE catalog; 
postgres=# CREATE USER catalog;
postgres=# ALTER ROLE catalog WITH PASSWORD '123456';
postgres=# GRANT ALL PRIVILEGES ON DATABASE catalog TO catalog;
postgres=# \q    # quit postgreq
exit ... Exit from user "postgres"
```
- Back to the 'grader' username and install git with `$ sudo apt-get install git`

- Create 'catalog' folder in **/var/www** with `$ sudo touch /var/www/catalog`

# Deploy the Item Catalog project
- To clone the project into /var/www/catalog, `$ sudo git clone https://github.com/can10/python-flask-app.git catalog`

- Change all the DB-related code in the source code from ```python engine = create_engine('sqlite:///catalog.db') ``` to ```python
engine = create_engine('postgresql://catalog:123456@localhost/catalog')```

- Change in the source code the client_secrets.json file path from `client_secrets.json` line to `/var/www/catalog/catalog/client_secrets.json`

- Change the name of the main file into `__init__.py` with `$ sudo mv application.py __init__.py`

- Install all the necessary packages,
```
$ sudo apt-get install python-pip
$ sudo pip install Flask
$ sudo pip install sqlalchemy
$ sudo pip install psycopg2
$ sudo pip install oauth2client
$ sudo pip install httplib2
$ sudo pip install requests
```

- `$ sudo nano /etc/apache2/sites-available/catalog.conf`
```
<VirtualHost *:80>
    ServerName 34.219.250.55
    ServerAlias http://ec2-34-219-250-55.us-west-2.compute.amazonaws.com
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

- Come back to **/var/www/catalog** and create catalog.wsgi
```bash
$ cd /var/www/catalog
$ sudo nano catalog.wsgi
```

- Add the source code:
```python
#!/usr/bin/python 
import sys 
import logging 

logging.basicConfig(stream=sys.stderr) 
sys.path.insert(0,"/var/www/catalog/")  

from catalog import app as application 
application.secret_key = 'super_secret_key'
```

- `$ sudo a2ensite catalog`     # enable
- `$ ls /etc/apache2/sites-enabled`    # this displays the sites enabled
- `$ ls /etc/apache2/sites-available`   # this displays the sites available to use

- `$ service apache2 reload`

- Test: `http://ec2-34-219-250-55.us-west-2.compute.amazonaws.com` - You should see the catalog application!

**Google sign-in part:**
- Add these two entries into JS origins:
```
http://34.219.250.55
http://ec2-34-219-250-55.us-west-2.compute.amazonaws.com
```

- Add this entry to Authorized redirect URIs
```
http://ec2-34-219-250-55.us-west-2.compute.amazonaws.com/gconnect
```

- Download the JSON credentials and copy paste the content of it into `client_secrets.json` in Linux.

- Finish! you can now access the catalog web application with:
`http://ec2-34-219-250-55.us-west-2.compute.amazonaws.com`

```bash
/var/www/catalog
    |-- catalog.wsgi
    |__ /catalog
         |-- __init__.py
         |-- client_secrets.json
         |-- catalog_database_setup.py
         |-- catalog_database_data.py
         |-- catalog_database_data_delete.py
         |-- /static
         |-- /templates
```
