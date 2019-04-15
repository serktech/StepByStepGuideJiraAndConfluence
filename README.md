# StepByStepGuideJiraAndConfluence

## A Step-by-Step Guide: How to install / Installing Jira applications on Linux( CentOS 7 - 7.6.1810 ) Atlassian Jira Software and Confluence, Postgres, SSL. Project Management Workflow 2019. 

In this guide we'll run you through installing a Jira application in a production environment, with an external database, using the Linux installer.

This is the most straightforward way to get your production site up and running on a Linux server.

1. Server configuration Step Guide: 
StepGuide-JiraAndConfluence.md

2. Documentation Source for part 2: Installing Jira applications on Linux - Server 8.1 
https://confluence.atlassian.com/adminjiraserver/installing-jira-applications-on-linux-938846841.html

* This is a short crib for installing Confluence and Jira on CentOS 7 *

Source: 
http://valynkin.ru/ustanovka-atlassian-confluence-i-jira.html

https://extremeshok.com

## Confluence Step Guide

### Install Postgres
```
yum install postgresql-server
sudo su - postgres -c "initdb -E UTF8 -D '/var/lib/pgsql/data'"
systemctl enable postgresql
```

Add to the /var/lib/pgsql/data/postgresql.conf file:
```
client_encoding = UTF8
```
  
Configuring password access:
```
vi /var/lib/pgsql/data/pg_hba.conf
```
  
should be like this ( host .. md5 ):
```
host all all 127.0.0.1/32 md5
host all all :: 1/128 md5
```
  
### Start postgres
```
systemctl start postgresql
```
  
Create username and database
```
sudo -u postgres createuser --no-password --no-createdb --no-superuser --no-createrole wiki
sudo -u postgres psql -c "ALTER USER wiki WITH PASSWORD 'Pa$$word';"
sudo -u postgres psql -c "CREATE DATABASE wiki WITH OWNER wiki ENCODING 'UTF8' TEMPLATE = template0;"
```
  
### Download the distribution from the site and install

*(steps from Attlasian guide)*

Watch logs:
```
tail -f /opt/atlassian/confluence/logs/catalina.out
tail -f /var/atlassian/application-data/confluence/logs/atlassian-confluence.log
```
  
Rollback in case of installation errors:
```
/etc/init.d/confluence stop
 sudo -u postgres dropdb wiki
 sudo -u postgres createdb wiki --encoding = UTF8 --template=template0 --owner=wiki
 rm -rf /var/atlassian/application-data/confluence/
 mkdir -p /var/atlassian/application-data/confluence/
 chown -R wiki:wiki/var/atlassian/application-data/confluence/
 /etc/init.d/confluence start
```
  
## Jira

### Install Postgres

```
yum install postgresql-server
postgresql-setup initdb
systemctl enable postgresql
```
  
Add to the /var/lib/pgsql/data/postgresql.conf file:
```
client_encoding = UTF8
```

Configuring password access:
```
vi /var/lib/pgsql/data/pg_hba.conf
```
should be like this ( host .. md5 ):
```
host    all     all     127.0.0.1/32    md5
host    all     all     :: 1/128        md5
```
  
### Start Postgres

```
systemctl start postgresql
```

Create user and database
```
sudo -u postgres createuser --no-password --no-createdb --no-superuser --no-createrole djira
 sudo -u postgres psql -c "ALTER USER djira WITH PASSWORD 'Pa$$word';"
 sudo -u postgres psql -c "CREATE DATABASE djira WITH OWNER djira ENCODING 'UTF8' TEMPLATE = template0;"
```
  
### Download the distribution from the site and install

After installation, need to add code in the beginning of the 
```
/etc/init.d/jira
```
startup script, otherwise there will be question marks instead of Russian characters after the reboot:
*optional for Russian characters*

```
export JAVA_OPTS = "- Dfile.encoding = UTF-8 -Dsun.jnu.encoding = UTF-8"
```
  
Watch logs:

```
tail -f /opt/atlassian/jira/logs/catalina.out
tail -f /var/atlassian/application-data/jira/log/atlassian-jira.log
```
  
Rollback in case of installation errors:

```
/etc/init.d/jira stop
 sudo -u postgres dropdb djira
 sudo -u postgres createdb djira --encoding = UTF8 --template=template0 --owner=djira
 rm -rf /var/atlassian/application-data/jira/
 mkdir -p /var/atlassian/application-data/jira/
 chown -R jira:jira/var/atlassian/application-data/jira/
 /etc/init.d/jira start
```
  
## Nginx reverse proxy with SSL

### Generate a self-signed ssl certificate

```
mkdir /etc/nginx/ssl
openssl req -x509 -nodes -days 9999 -newkey rsa: 2048 -keyout /etc/nginx/ssl/qqode.key -out /etc/nginx/ssl/qqode.crt
```
  
Nginx config:

```
 server {
    listen 80;
    server_name jira.yourdomain.ru;
    rewrite / https://$http_host$uri permanent;

    access_log /var/log/nginx/jira.access.log;
    error_log  /var/log/nginx/jira.error.log;
 }
 server {
     listen 443 ssl;
    server_name jira.yourdomain.com;

    ssl_certificate      /etc/nginx/ssl/qqode.crt;
    ssl_certificate_key  /etc/nginx/ssl/qqode.key;
    ssl_protocols        SSLv2 SSLv3 TLSv1 TLSv1.1 TLSv1.2;
    ssl_session_cache shared:SSL:3m;
    ssl_session_timeout  10m;

    access_log /var/log/nginx/jira.access.log;
    error_log  /var/log/nginx/jira.error.log;

    proxy_set_header Host $http_host;
    proxy_set_header    X-Real-IP       $remote_addr;
    proxy_set_header    X-Forwarded-For $remote_addr;

    location / {
        proxy_pass http://192.9.200.16:8081;
    }
 }
 server {
     listen 80;
    server_name wiki.yourdomain.com;

    rewrite / https://$http_host$uri permanent;

    access_log /var/log/nginx/wiki.access.log;
    error_log  /var/log/nginx/wiki.error.log;
 }

 server {
     listen 443 ssl;
    server_name wiki.yourdomain.com;

    ssl_certificate      /etc/nginx/ssl/qqode.crt;
    ssl_certificate_key  /etc/nginx/ssl/qqode.key;
    ssl_protocols        SSLv2 SSLv3 TLSv1 TLSv1.1 TLSv1.2;
    ssl_session_cache shared:SSL:3m;
    ssl_session_timeout  10m;

    access_log /var/log/nginx/wiki.access.log;
    error_log  /var/log/nginx/wiki.error.log;

    proxy_set_header Host $http_host;
    proxy_set_header    X-Real-IP       $remote_addr;
    proxy_set_header    X-Forwarded-For $remote_addr;

    location / {
        proxy_pass http://192.9.200.17:8091/;
    }
 }
```

Edit server.xml files ( 
/opt/atlassian/jira/conf/server.xml
and
/opt/atlassian/confluence/conf/server.xml
). 
We need to add a connector that will accept ssl connections.

Jira:

```
<Connector port = "8081"
                maxThreads = "150"
                minSpareThreads = "25"
                connectionTimeout = "20000"
                enableLookups = "false"
                maxHttpHeaderSize = "8192"
                protocol = "HTTP / 1.1"
                useBodyEncodingForURI = "true"
                redirectPort = "8443"
                acceptCount = "100"
                disableUploadTimeout = "true"
                scheme = "https" proxyName = "jira.yourdomain.com" proxyPort = "443" secure = "true" />
```
  
Confluence:

```
<Connector port = "8091"
            maxThreads = "48"
            minSpareThreads = "10"
            connectionTimeout = "20000"
            enableLookups = "false"
            maxHttpHeaderSize = "8192"
            protocol = "HTTP / 1.1"
            useBodyEncodingForURI = "true"
            redirectPort = "8443"
            acceptCount = "10"
            URIEncoding = "UTF-8"
            disableUploadTimeout = "true"
            scheme = "https" proxyName = "wiki.yourdomain.com" proxyPort = "443" secure = "true" />
```
  
