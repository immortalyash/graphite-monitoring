Getting Started
=
###System
```
Nginx
Supervisord
Gunicorn
Postgresql
Graphite
Collectd
Grafana
```

###Install Dependencies
Core dependencies
```
sudo apt-get -y install build-essential python-dev python-setuptools aptitude
sudo easy_install pip
sudo pip install virtualenv virtualenvwrapper
```
**Note-** Add these lines to ~/.bashrc for virtualenvwrapper and restart terminal
```
# Virtual Environment
export WORKON_HOME=$HOME/.virtualenvs
export PROJECT_HOME=$HOME/Devel
source /usr/local/bin/virtualenvwrapper.sh

# Graphite
export GRAPHITE_ROOT=/opt/graphite
```

Create your own Virtual Environment and work on
```
mkvirtualenv monitor
workon monitor
```

Nginx and Supervisor dependencies
```
sudo apt-get -y install nginx supervisor
```
**Note-** Ngins and Supervisor config files inside conf folder respectively

Database dependencies
```
sudo apt-get -y install postgresql postgresql-contrib postgresql-client pgadmin3 libpq-dev
```

Monitoring dependencies
```
sudo apt-get -y install collectd collectd-utils \
    adduser libfontconfig
```

App Dependencies
```
pip install -r requirements.txt
```

Install graphite-web and carbon separately with sudo (run both commands separately)
```
sudo pip install graphite-web
sudo pip install carbon
```

###Monitoring Config
Sync Database and manage settings of graphite, first copy local_settings.py as template
```
sudo cp local_settings.py.example local_settings.py
sudo vim /opt/graphite/webapp/graphite/local_settings.py
```
**Note-** Change SECRET_KEY, ALLOWED_HOSTS, TIME_ZONE & DEBUG according to your config

Add these static files root in nginx
```
cd /etc/nginx/sites-available/
sudo vim monitor.conf
```

Add these lines and save
```
location /static/ {
    alias /opt/graphite/static/;
}
```

Config Carbon
```
pushd /opt/graphite/conf
sudo cp carbon.conf.example carbon.conf
sudo cp storage-schemas.conf.example storage-schemas.conf
sudo cp storage-aggregation.conf.example storage-aggregation.conf
```

Enable ENABLE_LOGROTATION and add USER for Carbon
```
sudo vim /opt/graphite/conf/carbon.conf
```
change ENABLE_LOGROTATION to **True**

Storage Schema for monitoring
```
sudo vim /opt/graphite/conf/storage-schemas.conf
```

Add carbon to start at boot
```
sudo cp conf/init.d/carbon-cache /etc/init.d/carbon-cache
sudo chmod +x /etc/init.d/carbon-cache
sudo update-rc.d carbon-cache defaults 90 10
sudo systemctl daemon-reload
```

To start/stop carbon service
```
sudo service carbon-cache stop
sudo service carbon-cache start
```

###Create Database

Switch to postgres user before proceeding
```
 sudo su - postgres
```

Switch to postgresql command line interface
```
 psql
```

Create Database
```
 CREATE DATABASE dbname;
```

Create user for our database
```
 CREATE USER username WITH PASSWORD 'password';
```

Switch to our new database and grant privileges to our user
```
 \c dbname
```

Granting all privileges to our new user
```
 GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO username;
 GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO username;
 GRANT ALL PRIVILEGES ON DATABASE dbname TO username;
```

Exit Postgresql command line
```
 \q
```

Exit user
```
 exit
```

###Add Database details in django

Open graphite settings.py
```
 sudo vim /opt/graphite/webapp/graphite/local_settings.py
```

Find DATABASE variable and change details in default to below
```
 'ENGINE': 'django.db.backends.postgresql_psycopg2',
 'NAME': 'dbname',
 'USER': 'username',
 'PASSWORD': 'password',
 'HOST': 'localhost'
```

To migrate tables in database
```
PYTHONPATH=$GRAPHITE_ROOT/webapp django-admin.py migrate --settings=graphite.settings
```

###Configure Collectd
```
sudo vim /etc/collectd/collectd.conf
```

Change following according to your needs
```
Hostname "example.com"
```

Load plugins to your needs
```
LoadPlugin cpu
LoadPlugin df
LoadPlugin entropy
LoadPlugin interface
LoadPlugin load
LoadPlugin memory
LoadPlugin nginx
LoadPlugin processes
LoadPlugin rrdtool
LoadPlugin users
LoadPlugin write_graphite
```

Configure the plugins loaded
___
df
```
<Plugin df>
    Device "/dev/root"
    MountPoint "/"
    FSType "ext4"
</Plugin>
```

interface
```
<Plugin interface>
    Interface "eth0"
    IgnoreSelected false
</Plugin>
```

nginx (nginx rule also to be added)
```
<Plugin "nginx">
    URL "https://example.com/nginx_status"
    CACert "/path/to/cert/fullchain.pem"
</Plugin>
```

nginx rule in /etc/nginx/sites-available/example.conf
```
location /nginx_status {
    stub_status on;
    access_log off;
    allow 127.0.0.1;
    deny all;
}
```

write_graphite
```
<Plugin write_graphite>
    <Node "example.com">
        Host "localhost"
        Port "2003"
        Protocol "tcp"
        LogSendErrors true
        Prefix "collectd."
        StoreRates true
        AlwaysAppendDS false
        EscapeCharacter "_"
    </Node>
</Plugin>
```

Add Collectd in graphite storage-schema for data to be logged
```
sudo vim /etc/carbon/storage-schemas.conf
```

Add these lines below carbon block
```
[collectd]
pattern = ^collectd.*
retentions = 10s:1d,1m:7d,10m:1y
```

Restart the process in sequence
```
sudo service carbon-cache stop  ## Wait for few seconds
sudo service carbon-cache start
sudo service collectd stop
sudo service collectd start
```

###Installing Grafana
Install grafana for ARM from packages folder or compile from source
```
sudo dpkg -i packages/grafana*.dpkg
```

Create new separate database for grafana and add the details in grafana.ini
```
sudo vim /etc/grafana/grafana.ini
```

To start/stop Grafana server(runs on default port 3000)
```
sudo service grafana-server stop
sudo service grafana-server start
```

**Note-** You would require a reverse proxy to grafana for access via nginx, nginx settings in conf/nginx/monitor.conf

**Note-** default username and password is admin

## TODO
* statsd - https://www.digitalocean.com/community/tutorials/how-to-configure-statsd-to-collect-arbitrary-stats-for-graphite-on-ubuntu-14-04
