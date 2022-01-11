# Installation instructions for CentOS 7

## Assumptions
elk-server.hspace.com __(ELK master)__

elk-client.hspace.com __(client machine)__

## ELK Stack installation on server.example.com
###### Install Java 8
```
yum install -y java-1.8.0-openjdk
```
###### Import PGP Key
```
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```
###### Create Yum repository
```
cat >>/etc/yum.repos.d/elk.repo<<EOF
[elasticsearch]
name=Elasticsearch repository for 7.x packages
baseurl=https://artifacts.elastic.co/packages/7.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF
```
### Elasticsearch
###### Install Elasticsearch
```
sudo yum install -y --enablerepo=elasticsearch elasticsearch
```
###### Enable and start elasticsearch service
```
systemctl daemon-reload
systemctl enable elasticsearch
systemctl start elasticsearch
systemctl status elasticsearch
```
###### Elasticsearch Configuration files
```
rpm -qc elasticsearch
```
###### Elasticsearch Log files Location
```
ls /var/log/elasticsearch/

journalctl --unit elasticsearch
```


### Kibana
###### Import GPG Key
```
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```
###### Create Yum repository
```
cat >>/etc/yum.repos.d/kibana.repo<<EOF
[kibana-7.x]
name=Kibana repository for 7.x packages
baseurl=https://artifacts.elastic.co/packages/7.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF
```

###### Install kibana
```
sudo yum install -y kibana
```
###### Enable and start kibana service
```
systemctl daemon-reload
systemctl enable kibana
systemctl start kibana
systemctl status kibana
```
###### Kibana Configuration Files
```
rpm -qc kibana
```
###### Kibana Logs
```
journalctl --unit kibana
```

###### Install Nginx
```
yum install -y epel-release
yum install -y nginx
```
###### Create Proxy configuration
Remove server block from the default config file /etc/nginx/nginx.conf
And create a new config file
```
cat >>/etc/nginx/conf.d/kibana.conf<<EOF
server {
    listen 80;
    server_name elk-server.hspace.com;
    location / {
        proxy_pass http://localhost:5601;
    }
}
EOF
```
###### Enable and start nginx service
```
systemctl enable nginx
systemctl start nginx
```

### Logstash
###### Import GPG Key
```
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```
###### Create Yum repository
```
cat >>/etc/yum.repos.d/logstash.repo<<EOF
[logstash-7.x]
name=Elastic repository for 7.x packages
baseurl=https://artifacts.elastic.co/packages/7.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF
```

###### Install logstash
```
yum install -y logstash
```
###### Generate SSL Certificates
```
openssl req -subj '/CN=elk-server.hspace.com/' -x509 -days 3650 -nodes -batch -newkey rsa:2048 -keyout /etc/pki/tls/private/logstash.key -out /etc/pki/tls/certs/logstash.crt
```
###### Create Logstash config file
To configure Logstash, you create a config file that specifies which plugins you want to use and settings for each plugin. You can reference event fields in a configuration and use conditionals to process events when they meet certain criteria.
```
vi /etc/logstash/conf.d/01-logstash-simple.conf
```
Paste the below content
```
input {
  beats {
    port => 5044
    ssl => true
    ssl_certificate => "/etc/pki/tls/certs/logstash.crt"
    ssl_key => "/etc/pki/tls/private/logstash.key"
  }
}

filter {
    if [type] == "syslog" {
        grok {
            match => {
                "message" => "%{SYSLOGTIMESTAMP:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{DATA:syslog_program}(?:\[%{POSINT:syslog_pid}\])?: %{GREEDYDATA:syslog_message}"
            }
            add_field => [ "received_at", "%{@timestamp}" ]
            add_field => [ "received_from", "%{host}" ]
        }
        syslog_pri { }
        date {
            match => [ "syslog_timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
        }
    }
}

output {
    elasticsearch {
        hosts => "localhost:9200"
        index => "%{[@metadata][beat]}-%{+YYYY.MM.dd}"
    }
}
```
###### Enable and Start logstash service
```
systemctl enable logstash
systemctl start logstash
systemctl status logstash
```
###### To check the logs
```
journalctl --unit logstash
```

## FileBeat installation on client.example.com
###### Create Yum repository
```
cat >>/etc/yum.repos.d/elk.repo<<EOF
[ELK-6.x]
name=ELK repository for 6.x packages
baseurl=https://artifacts.elastic.co/packages/6.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF
```
###### Install Filebeat
```
yum install -y filebeat
```
###### Copy SSL certificate from server.example.com
```
scp server.example.com:/etc/pki/tls/certs/logstash.crt /etc/pki/tls/certs/
```
###### Configure Filebeat 
[Refer my youtube video]
###### Enable and start Filebeat service
```
systemctl enable filebeat
systemctl start filebeat
```
### Configure Kibana Dashboard
All done. Now you can head to Kibana dashboard and add the index.


