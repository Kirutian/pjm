adduser username
passwd username
usermod -aG wheel username


sudo yum install -y epel-release
sudo yum install -y wget

sudo yum install -y ntp
sudo timedatectl set-timezone America/Denver
sudo firewall-cmd --add-service=ntp --permanent
sudo firewall-cmd --reload
sudo systemctl start ntpd
sudo systemctl enable ntpd
sudo systemctl status ntpd
sudo yum -y update

REBOOT LOG IN AS username

sudo yum install nginx
sudo systemctl start nginx
sudo firewall-cmd --permanent --zone=public --add-service=http 
sudo firewall-cmd --permanent --zone=public --add-service=https
sudo firewall-cmd --reload
sudo systemctl enable nginx
sudo yum install certbot python2-certbot-nginx

sudo certbot --nginx -d example.com -d www.example.com


Install PostgreSQL
sudo yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo yum -y install postgresql12-server postgresql12-contrib postgresql12 
sudo /usr/pgsql-12/bin/postgresql-12-setup initdb
sudo systemctl start postgresql-12
sudo systemctl enable postgresql-12
sudo passwd postgres
sudo su - postgres
createuser taiga 
psql
ALTER USER taiga WITH ENCRYPTED password 'NEWPASSWORD';
CREATE DATABASE taiga OWNER taiga;
\q
exit

Install Python 3
sudo yum -y install gcc autoconf flex bison libjpeg-turbo-devel freetype-devel zlib-devel zeromq3-devel gdbm-devel ncurses-devel automake libtool libffi-devel curl git tmux libxml2-devel libxslt-devel openssl-devel gcc-c++
sudo cd ~
wget https://www.python.org/ftp/python/3.7.4/Python-3.7.4.tgz
tar xf Python-3.7.4.tgz
cd Python-3.7.4
./configure --enable-optimizations --prefix=/usr
sudo make altinstall
sudo pip3.7 install virtualenv virtualenvwrapper
sudo pip3.7 install --upgrade setuptools pip 

Install RabbitMQ
curl -s https://packagecloud.io/install/repositories/rabbitmq/rabbitmq-server/script.rpm.sh | sudo bash
curl -s https://packagecloud.io/install/repositories/rabbitmq/erlang/script.rpm.sh | sudo bash
sudo rpm --import https://packagecloud.io/rabbitmq/rabbitmq-server/gpgkey
sudo yum install -y rabbitmq-server
sudo yum install -y erlang
sudo systemctl start rabbitmq-server
sudo systemctl enable rabbitmq-server
sudo rabbitmqctl add_user taiga SUPERPASSWORD
sudo rabbitmqctl add_vhost taiga
sudo rabbitmqctl set_permissions -p taiga taiga ".*" ".*" ".*"
cd ~
Install Nodejs
curl -sL https://rpm.nodesource.com/setup_8.x | sudo -E bash -
sudo yum install -y nodejs pwgen
sudo npm install -g coffee-script gulp
sudo useradd -s /bin/bash taiga
sudo su - taiga
mkdir -p ~/logs
git clone https://github.com/taigaio/taiga-back.git taiga-back
cd taiga-back
git checkout stable
echo "VIRTUALENVWRAPPER_PYTHON='/bin/python3.7'" >> ~/.bashrc
echo "source /usr/bin/virtualenvwrapper.sh" >> ~/.bashrc
source ~/.bashrc
mkvirtualenv -p /bin/python3.7 taiga
pip3.7 install --upgrade setuptools
pip3.7 install -r requirements.txt
python3.7 manage.py migrate --noinput
python3.7 manage.py loaddata initial_user
python3.7 manage.py loaddata initial_project_templates
python3.7 manage.py compilemessages
python3.7 manage.py collectstatic --noinput
pwgen -s -1 64
[crypted string]

vi ~/taiga-back/settings/local.py

from .common import *

MEDIA_URL = "https://subdomain.domain.com/media/"
STATIC_URL = "https://subdomain.domain.com/static/"
SITES["front"]["scheme"] = "https"
SITES["front"]["domain"] = "subdomain.domain.com"

SECRET_KEY = "[crypted string]"

DEBUG = False
PUBLIC_REGISTER_ENABLED = True

DEFAULT_FROM_EMAIL = "mail@example.com"
SERVER_EMAIL = DEFAULT_FROM_EMAIL

#CELERY_ENABLED = True

EVENTS_PUSH_BACKEND = "taiga.events.backends.rabbitmq.EventsPushBackend"
EVENTS_PUSH_BACKEND_OPTIONS = {"url": "amqp://taiga:SUPERPASSWORD@localhost:5672/taiga"}

# Uncomment and populate with proper connection parameters
# for enable email sending. EMAIL_HOST_USER should end by @domain.tld
#EMAIL_BACKEND = "django.core.mail.backends.smtp.EmailBackend"
#EMAIL_USE_TLS = False
#EMAIL_HOST = "mail.example.com"
#EMAIL_HOST_USER = "mail@example.com"
#EMAIL_HOST_PASSWORD = "SMTPPassword"
#EMAIL_PORT = 25

# Uncomment and populate with proper connection parameters
# for enable github login/singin.
#GITHUB_API_CLIENT_ID = "yourgithubclientid"
#GITHUB_API_CLIENT_SECRET = "yourgithubclientsecret"


FRONT END
cd ~
git clone https://github.com/taigaio/taiga-front-dist.git taiga-front-dist
cd taiga-front-dist
git checkout stable
vi ~/taiga-front-dist/dist/conf.json
{
    "api": "https://subdomain.domain.com/api/v1/",
    "eventsUrl": "wss://subdomain.domain.com/events",
    "eventsMaxMissedHeartbeats": 5,
    "eventsHeartbeatIntervalTime": 60000,
    "eventsReconnectTryInterval": 10000,
    "debug": true,
    "debugInfo": false,
    "defaultLanguage": "en",
    "themes": ["taiga"],
    "defaultTheme": "taiga",
    "publicRegisterEnabled": true,
    "feedbackEnabled": true,
    "privacyPolicyUrl": null,
    "termsOfServiceUrl": null,
    "maxUploadFileSize": null,
    "contribPlugins": [],
    "tribeHost": null,
    "importers": [],
    "gravatar": true
}

cd ~
git clone https://github.com/taigaio/taiga-events.git taiga-events
cd taiga-events
npm install
vi ~/taiga-events/config.json

{
    "url": "amqp://taiga:SUPERPASSWORD@localhost:5672/taiga",
    "secret": "[crypted string]",
    "webSocketServer": {
        "port": 8888
    }
}

exit
sudo pip3.7 install circus
sudo mkdir /etc/circus
sudo mkdir /etc/circus/conf.d
sudo vi /etc/circus/circus.ini
[circus]
check_delay = 5
endpoint = tcp://127.0.0.1:5555
pubsub_endpoint = tcp://127.0.0.1:5556
include = /etc/circus/conf.d/*.ini

sudo vi /etc/circus/conf.d/taiga.ini

[watcher:taiga]
working_dir = /home/taiga/taiga-back
cmd = gunicorn
args = -w 3 -t 60 --pythonpath=. -b 127.0.0.1:8001 taiga.wsgi
uid = taiga
numprocesses = 1
autostart = true
send_hup = true
stdout_stream.class = FileStream
stdout_stream.filename = /home/taiga/logs/gunicorn.stdout.log
stdout_stream.max_bytes = 10485760
stdout_stream.backup_count = 4
stderr_stream.class = FileStream
stderr_stream.filename = /home/taiga/logs/gunicorn.stderr.log
stderr_stream.max_bytes = 10485760
stderr_stream.backup_count = 4

[env:taiga]
PATH = /home/taiga/.virtualenvs/taiga/bin:$PATH
TERM=rxvt-256color
SHELL=/bin/bash
USER=taiga
LANG=en_US.UTF-8
HOME=/home/taiga
PYTHONPATH=/home/taiga/.virtualenvs/taiga/lib/python3.7/site-packages

sudo vi /etc/circus/conf.d/taiga-events.ini

[watcher:taiga-events]
working_dir = /home/taiga/taiga-events
cmd = /usr/bin/coffee
args = index.coffee
uid = taiga
numprocesses = 1
autostart = true
send_hup = true
stdout_stream.class = FileStream
stdout_stream.filename = /home/taiga/logs/taigaevents.stdout.log
stdout_stream.max_bytes = 10485760
stdout_stream.backup_count = 12
stderr_stream.class = FileStream
stderr_stream.filename = /home/taiga/logs/taigaevents.stderr.log
stderr_stream.max_bytes = 10485760
stderr_stream.backup_count = 12

sudo vi /etc/systemd/system/circus.service
[Unit]
Description=Circus process manager
After=syslog.target network.target nss-lookup.target
[Service]
Type=simple
ExecReload=/usr/bin/circusctl reload
ExecStart=/usr/bin/circusd /etc/circus/circus.ini
Restart=always
RestartSec=5

[Install]
WantedBy=default.target

sudo systemctl start circus
sudo systemctl enable circus

sudo vi /etc/nginx/conf.d/taiga.conf

server {
    listen 80;
    server_name taiga.example.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;
    server_name taiga.example.com;

    access_log /home/taiga/logs/nginx.access.log;
    error_log /home/taiga/logs/nginx.error.log;

    large_client_header_buffers 4 32k;
    client_max_body_size 50M;
    charset utf-8;

    index index.html;

    # Frontend
    location / {
        root /home/taiga/taiga-front-dist/dist/;
        try_files $uri $uri/ /index.html;
    }

    # Backend
    location /api {
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Scheme $scheme;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://127.0.0.1:8001/api;
        proxy_redirect off;
    }

    location /admin {
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Scheme $scheme;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://127.0.0.1:8001$request_uri;
        proxy_redirect off;
    }

    # Static files
    location /static {
        alias /home/taiga/taiga-back/static;
    }

    # Media files
    location /media {
        alias /home/taiga/taiga-back/media;
    }

     location /events {
        proxy_pass http://127.0.0.1:8888/events;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_connect_timeout 7d;
        proxy_send_timeout 7d;
        proxy_read_timeout 7d;
    }

    add_header Strict-Transport-Security "max-age=63072000; includeSubdomains; preload";
    add_header Public-Key-Pins 'pin-sha256="klO23nT2ehFDXCfx3eHTDRESMz3asj1muO+4aIdjiuY="; pin-sha256="633lt352PKRXbOwf4xSEa1M517scpD3l5f79xMD9r9Q="; max-age=2592000; includeSubDomains';

    ssl on;
    ssl_certificate /etc/letsencrypt/live/taiga.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/taiga.example.com/privkey.pem;
    ssl_session_timeout 5m;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!3DES:!MD5:!PSK';
    ssl_session_cache shared:SSL:10m;
    ssl_dhparam /etc/ssl/dhparam.pem;
    ssl_stapling on;
    ssl_stapling_verify on;

}

sudo systemctl restart nginx
sudo systemctl status nginx

TROUBLESHOOTING:
sudo chown -R taiga:taiga /home/taiga/
sudo chmod o+x /home/taiga/

sudo setenforce 0
sudo setsebool -P httpd_read_user_content 1
