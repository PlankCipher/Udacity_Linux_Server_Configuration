# Linux server configuration

**Public IP address** : ~~35.178.235.41~~

**SSH port** : 2200

**URL** : [~~http://ec2-35-178-235-41.eu-west-2.compute.amazonaws.com/~~](http://ec2-35-178-235-41.eu-west-2.compute.amazonaws.com/)

## Configurations made:

### 1. Update all currently installed packages.

```shell
$ sudo apt-get update
$ sudo apt-get upgrade
$ sudo apt-get dist-upgrade
```

### 2. Create a new user `grader` and grant it sudo permissions

```shell
$ sudo adduser grader
$ sudo nano /etc/sudoers.d/grader # Fill this file with "grader ALL=(ALL:ALL) ALL" to give "grader" user sudo permissions
```

### 3. Configure the local timezone to UTC

```shell
$ sudo dpkg-reconfigure tzdata # Configure it to UTC
$ sudo apt-get install ntp # Install ntp for a better sync. of the server's time
```

### 4. Configure the SSH key pair that will be used later to login to `grader`

```shell
$ ssh-keygen -f /root/.ssh/grader.rsa # On the local machine
$ sudo mkdir /home/grader/.ssh/ # On the server
$ sudo touch /home/grader/.ssh/authorized_keys # On the server
$ sudo nano /home/grader/.ssh/authorized_keys # Fill it with the content of the public key created on the local machine, "grader.rsa.pub" in my case.
$ sudo chmod 700 /home/grader/.ssh                  # Give proper permissions for ".ssh" directory
$ sudo chmod 644 /home/grader/.ssh/authorized_keys  # Give proper permissions for "authorized_keys" file
$ sudo chown -R grader:grader /home/grader/.ssh     # Change the owner to "grader"
```

### 5. Enforce Key-based `SSH` authentication

```shell
$ sudo nano /etc/ssh/sshd_config # Make sure that "PasswordAuthentication" line is set to "no"
$ sudo service ssh restart
```

### 6. Change the `SSH` port from **22** to **2200**

```shell
$ sudo nano /etc/ssh/sshd_config # Edit the Port line to 2200 (NOTE: If needed, remove the hashtag "#" symbol infront of it)
$ sudo service ssh restart
```

### 7. Disable remote login of the `root` user

```shell
$ sudo nano /etc/ssh/sshd_config # Make sure that "PermitRootLogin" line is set to "no"
$ sudo service ssh restart
```

### 8. Configure the Uncomplicated Firewall (UFW)

```shell
$ sudo ufw allow 2200/tcp   # For the SSH port 2200
$ sudo ufw allow 80/tcp     # For the HTTP port 80
$ sudo ufw allow 123/udp    # For the NTP port 123
$ sudo ufw enable
```

### 9. Install and configure Apache to serve a Python mod_wsgi application

```shell
$ sudo apt-get install apache2
$ sudo apt-get install python-setuptools libapache2-mod-wsgi
$ sudo service apache2 restart
```

### 10. Install git and clone the project

```shell
$ sudo apt-get install git
$ sudo a2enmod wsgi # Enable mod_wsgi
$ cd /var/www/
$ sudo mkdir catalog
$ cd catalog
$ sudo mkdir catalog
$ sudo git clone https://github.com/Mohammedayman25/catalog catalog
$ cd catalog
$ sudo mv project.py __init__.py # Rename the file that contains the flask application logic to __init__.py
```

### 11. Configure Apache to serve the flask application

```shell
$ sudo apt-get install python-dev
$ sudo apt-get install python-pip # Install pip installer
$ sudo pip install virtualenv
$ sudo virtualenv venv # Set virtual environment to name "venv"
$ sudo chmod -R 777 venv # Enable all permissions for the new virtual environment
$ source venv/bin/activate # Activate the virtual environment, commands after this command are executed inside the virtual environment
$ pip install Flask
$ python __init__.py # Run the app
$ deactivate # Deactivate the environment, commands after this command are executed outside the virtual environment
$ sudo nano /etc/apache2/sites-available/catalog.conf # Fill this file with the content of "catalog.conf" from this repo
$ sudo a2ensite catalog # Enable the virtual host
$ cd /var/www/catalog
$ sudo nano catalog.wsgi # Fill this file with the content of "catalog.wsgi" from this repo
$ sudo service apache2 restart
$ sudo nano .htaccess # Put this line "RedirectMatch 404 /\.git" inside this file to make the GitHub repo web inaccessible
$ source venv/bin/activate # Activate the virtual environment again
```

### 12. Install required Python modules

```shell
$ pip install httplib2
$ pip install requests
$ pip install --upgrade requests
$ pip install flask-seasurf
$ pip install --upgrade oauth2client
$ pip install sqlalchemy
$ sudo apt-get install python-psycopg2
```

### 13. Install Postgresql and configure it

```shell
$ sudo apt-get install postgresql postgresql-contrib
$ sudo adduser catalog # Make a user for psql
$ sudo su - postgres # Chanage to the user "postgres" that postgresql created
$ psql # Connect to the psql system
```

### 14. Create and configure the user/role catalog from psql to give the application a database

```sql
postgres=> CREATE USER catalog WITH PASSWORD 'udacity'; # Create the user/role catalog
postgres=> ALTER USER catalog CREATEDB;                 # Allow it to create databases
postgres=> CREATE DATABASE catalog WITH OWNER catalog;  # Create the database "catalog" for the owner "catalog" user
postgres=> \c catalog                                   # Connect to the database
postgres=> REVOKE ALL ON SCHEMA public FROM public;     # Revoke all rights
postgres=> GRANT ALL ON SCHEMA public TO catalog;       # Grant only access to the user/role catalog
postgres=> \q                                           # exit psql
```

### 15. Run the application
```shell
$ exit # Exit from the user postgres
$ sudo nano database_setup.py # Modify the engine declaration line to use postgresql (i.e. engine = create_engine('postgresql://catalog:udacity@localhost/catalog')), modify it also in "fill_database.py" and "__init__.py"
$ sudo nano __init__.py # Edit all the paths of "client_secrets.json" to the full path (i.e. /var/www/catalog/catalog/client_secrets.json)
$ sudo python3 database_setup.py # Create the database with its columns for the app
$ sudo python3 fill_database.py # Fill the database with data and items
$ sudo python3 __init__.py # Start the actual application
$ sudo service apache2 restart
```

### 16. Additional steps to get Google+ authentication working for the new URL

1. Go to the [project](https://console.developers.google.com/project) on the Developer Console.
2. Navigate to **`APIs & Services > OAuth consent screen > Edit App`**.
3. Add the server's domain under the **`Authorized domains`** section. (i.e. ec2-35-178-235-41.eu-west-2.compute.amazonaws.com)
4. Go to **`Credentials > {The project name}`**.
5. Add the server's domain under the **`Authorized JavaScript origins`**, prefixed by `http://`.
