# Redmine upgrade from 2.6.3 to 3.4 (latest stable) 

### Testing Stage - local machine

#### Container Diagram

```sequence
redmine depends_on: mysql
mysql will start first
mysql: keeps file on volumes
mysql: when it starts will create an empty db
```

```
$ docker-compose up -d mysql
Creating redmine_mysql ... 
Creating redmine_mysql ... done\
```


#### Steps on MYSQL Container
**Start only the mysql container**
```
[andra@naruto redmine.docker]$ docker exec -it redmine_mysql  bash
```
**On mysql container**
```
:/var/local# mysql -u redmine -p -P 13306 redmine < redmine_edw.sql
```

#### Start redmine container

```
$ docker-compose up -d
redmine_mysql is up-to-date
Creating redmine ... 
Creating redmine ... done
```

 





### Production- old machine migrate to new machine (from edw01 machine to artemis machine)


1. on edw01 machine  dump the database (we will need it to migrate the dump on artemis)

`# mysqldump -u redmine -p redmine_new > redmine_edw.sql`


2. Migrate `redmine_edw.sql` and `website files` from `edw01` to `artemis`
> (the following steps are executed on local machine)
```
scp user@edw01:/home/redmine_new.sql .
scp redmine_new.sql  edwsys@artemis.edw.ro:/home/edwsys
scp   user@edw01:/home/files.tar  .
scp files.tar  edwsys@artemis.edw.ro:/home/edwsys
```
```
vim /etc/hosts // add 88.99.2.80 sitename.ro…
```


3. Import the dump from mysql server on edw01 on redmine_mysql container
> on artemis machine
```
docker exec -it redmine_mysql bash
mysql -u redmine -p redmine < /var/local/redmine/backup/redmine_new.sql
```


4. Letsencript cert for artemis (on artemis machine)
```
letsencrypt certonly -d helpdesk.eaudeweb.ro
```

5. Add https
> or artemis machine
```
vim  /etc/httpd/conf.d/40-helpdesk.conf  
service httpd restart
```
```# cat /etc/httpd/conf.d/40-helpdesk.conf
```
```
<VirtualHost *:443>
        ServerName helpdesk.eaudeweb.ro

        AddDefaultCharset UTF-8

        SSLEngine on
        SSLCertificateFile "/etc/letsencrypt/live/helpdesk.eaudeweb.ro/cert.pem"
        SSLCertificateKeyFile "/etc/letsencrypt/live/helpdesk.eaudeweb.ro/privkey.pem"
        SSLCertificateChainFile "/etc/letsencrypt/live/helpdesk.eaudeweb.ro/chain.pem"
    
        RequestHeader set X_FORWARDED_PROTO 'https'
        SSLProtocol All -SSLv2 -SSLv3

        ProxyPass / http://localhost:8091/ retry=2
        ProxyPassReverse / http://localhost:8091/
        ProxyPreserveHost On

        TransferLog /var/log/httpd/helpdesk_error.log
        CustomLog /var/log/httpd/helpdesk.log combined
</VirtualHost>
<VirtualHost *:80>
    ServerAdmin anton@eaudeweb.ro
    ServerName helpdesk.eaudeweb.ro
        
    RewriteEngine On
    RewriteRule (.*) https://helpdesk.eaudeweb.ro$1
</VirtualHost>
<VirtualHost *:80>
    ServerAdmin anton@eaudeweb.ro
    ServerName redmine.eaudeweb.ro
        
    RewriteEngine On
    RewriteRule (.*) https://helpdesk.eaudeweb.ro$1
</VirtualHost>
<VirtualHost *:443>
        ServerName redmine.eaudeweb.ro

        SSLEngine on
        SSLCertificateFile "/etc/letsencrypt/live/redmine.eaudeweb.ro/cert.pem"
        SSLCertificateKeyFile "/etc/letsencrypt/live/redmine.eaudeweb.ro/privkey.pem"
        SSLCertificateChainFile "/etc/letsencrypt/live/redmine.eaudeweb.ro/chain.pem"

        SSLProtocol All -SSLv2 -SSLv3
        RewriteEngine On
        RewriteRule (.*) https://helpdesk.eaudeweb.ro$1
</VirtualHost>
```

6. on artemis 
`docker-compose up -d`

7. Test it in browser 


