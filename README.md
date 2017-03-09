# Server Side (Elastic Search and Kibana)
![AWS-EB-ELK](https://s10.postimg.org/8506gommh/aws_eb_elk_stack.png "AWS-EB-ELK")


# Create new (ec2 instance) on aws console dashboard.
#### Requirement: 
1. Amazon Linux AMI 2016.09.1
2. Elasticsearch 1.7.2 (2 Instance)
3. Kibana 4.1.2 (1 Instance)

# How to install
Command to install elasticsearch on ec2 instance (Elasticsearch 1.7.2). do this command to all elasticsearch instance

```sh
sudo su
yum update -y
cd /root
wget https://download.elastic.co/elasticsearch/elasticsearch/elasticsearch-1.7.2.noarch.rpm
yum install elasticsearch-1.7.2.noarch.rpm -y
rm -f elasticsearch-1.7.2.noarch.rpm
cd /usr/share/elasticsearch/
./bin/plugin -install mobz/elasticsearch-head
./bin/plugin -install lukas-vlcek/bigdesk
./bin/plugin install elasticsearch/elasticsearch-cloud-aws/2.7.1
./bin/plugin --install lmenezes/elasticsearch-kopf/1.5.7
```

# Configuration for elasticsearch
Edit elastic search configuration on: `vim /etc/elasticsearch/elasticsearch.yml`
```sh
cluster.name: company-log-elasticsearch
cloud.aws.access_key: AWS_KEY
cloud.aws.secret_key: SECRET_KEY
cloud.aws.region: ap-southeast-1
discovery.type: ec2
discovery.ec2.tag.Name: "company-log-elasticsearch"
http.cors.enabled: true
http.cors.allow-origin: "*"
```

# Restart the elasticsearch service
`service elasticsearch start`

# Create load balancer for 2 server and get the url
# (Optional) Security - elasticsearch can accessibly by local and vpn only.
IF YOU WANT THE ELASTIC SEARCH CAN ACCESSIBLY FROM LAN ONLY,
YOU HAVE SETUP 2 LOAD BALANCER (1 PUBLIC just allow office-public-ip)
AND (1 PRIVATE for host all application - logstash client)
AND YOU HAVE TO ADD THE INSTANCE TO LOAD BALANCER company-log-elasticsearch AND company-log-elasticsearch-pub
##### (Alternative) Security - create wide open security group

# Kibana 4.1.2
```sh
sudo su
yum update -y
cd /root
wget https://download.elastic.co/kibana/kibana/kibana-4.1.2-linux-x64.tar.gz
tar xzf kibana-4.1.2-linux-x64.tar.gz
rm -f kibana-4.1.2-linux-x64.tar.gz
```

# Edit kibana configuration
`vim /root/kibana-4.1.2-linux-x64/config/kibana.yml`
```sh
elasticsearch_url: "load_balancer_url"
```

# Start your kibana service and the kibana run on port 5601
```sh
nohup ./bin/kibana &
```

# Install the NGINX
```sh
# Install packages
yum install nginx httpd -y

# Create nginx user
sudo htpasswd -c /etc/nginx/htpasswd.users admin

# comment all server { listen 80 } on this file
vim /etc/nginx/nginx.conf

# add new configuration
vim /etc/nginx/conf.d/virtual.conf

server {
  listen 80;

  auth_basic "Restricted Access";
  auth_basic_user_file /etc/nginx/htpasswd.users;

  location / {
  proxy_pass http://localhost:5601;
  proxy_http_version 1.1;
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection 'upgrade';
  proxy_set_header Host $host;
  proxy_cache_bypass $http_upgrade;
  }
}
```

# Done ! you can access your kibana dashboard on `http://IP_PUBLIC/`
# And access the elasticsearch dashboard `http://your-elastic-url:9200/_plugin/head/`
# and bigdesk plugin elasticsearch at `http://your-elastic-url:9200/_plugin/bigdesk/`

# Client / Application Side (Logstash)

# Install LogStasher
`gem 'logstasher'`

# Add the ebscript on folder `.ebextensions`

##### 0_logstash_files.config
#
```sh
files:
  "/etc/yum.repos.d/logstash.repo":
    mode: "000644"
    owner: root
    group: root
    content: |
     [logstash-2.3]
     name=Logstash repository for 2.3.x packages
     baseurl=https://packages.elastic.co/logstash/2.3/centos
     gpgcheck=1
     gpgkey=https://packages.elastic.co/GPG-KEY-elasticsearch
     enabled=1
```

##### 1_rails_logstash.config
#
```sh
files:
  "/etc/logstash/conf.d/rails.conf":
    content: |
      input {
        file {
            path => "/var/app/current/log/logstasher.log"
            type => "rails_logs"
            codec =>   json {
              charset => "UTF-8"
            }
         }
      }

      filter {
        geoip {
          source => "ip"
          target => "geoip"
          database => "/etc/logstash/GeoLiteCity.dat"
          add_field => [ "[geoip][coordinates]", "%{[geoip][longitude]}" ]
          add_field => [ "[geoip][coordinates]", "%{[geoip][latitude]}"  ]
        }
      }

      output {
        elasticsearch {
          hosts => 'your_elastic_url'
          document_id => "%{request_id}"
          timeout => 15
        }
      }
```

##### 2_logstash.config
#
```sh
commands:
  create_post_dir:
    command: "mkdir /opt/elasticbeanstalk/hooks/appdeploy/post"
    ignoreErrors: true
  create_restartappserver_post_dir:
    command: "mkdir /opt/elasticbeanstalk/hooks/restartappserver/post"
    ignoreErrors: true
  100-install-logstash:
      command: "sudo yum -y install logstash"
files:
  "/opt/elasticbeanstalk/hooks/appdeploy/post/logstash_geoip.sh":
    mode: "000755"
    group: root
    owner: root
    content: |
      #!/bin/bash
      cd /etc/logstash
      rm GeoLiteCity*
      sudo curl -O "http://geolite.maxmind.com/download/geoip/database/GeoLiteCity.dat.gz"
      sudo gunzip GeoLiteCity.dat.gz

  "/opt/elasticbeanstalk/hooks/appdeploy/post/start_logstash.sh":
    mode: "000755"
    group: root
    owner: root
    content: |
      #!/bin/bash
      for pid in `ps aux | grep /etc/logstash/conf.d | grep -v grep | tr -s ' ' | cut -d ' ' -f 2`
      do
        sudo kill -9 $pid
      done

      export HOME=/var/lib/logstash

      sudo service logstash restart

  "/opt/elasticbeanstalk/hooks/restartappserver/post/start_logstash.sh":
    mode: "000755"
    group: root
    owner: root
    content: |
      #!/bin/bash
      for pid in `ps aux | grep /etc/logstash/conf.d | grep -v grep | tr -s ' ' | cut -d ' ' -f 2`
      do
        sudo kill -9 $pid
      done

      export HOME=/var/lib/logstash

      sudo service logstash restart
```

#Edit your ruby environment on `config/environments/develop-stage.rb` add this configuration:
#
```sh
  # Enable the logstasher logs for the current environment
  config.logstasher.enabled = true

  # Each of the following lines are optional. If you want to selectively disable log subscribers.
  config.logstasher.controller_enabled = false
  config.logstasher.mailer_enabled = false
  config.logstasher.record_enabled = false
  config.logstasher.view_enabled = false
  config.logstasher.job_enabled = false
```

# Create new config on folder `config/logstasher.yml`
#
```
controller_enabled: true
mailer_enabled: false
record_enabled: false
job_enabled: false
view_enabled: true
suppress_app_log: false
staging:
  enabled: true
  record_enabled: true
production:
  enabled: true
  mailer_enabled: true
  view_enabled: false
```

# and deploy your application with eb deploy


# AND ACCESS THE KIBANA AND ELASTICSEARCH DASHBOARD
KIBANA `http://IP_PUBLIC/`
ELASTICSEARCH PLUGIN HEAD `http://your-elastic-url:9200/_plugin/head/` 
ELASTICSEARCH PLUGIN BIGDESK `http://your-elastic-url:9200/_plugin/bigdesk/`

![AWS-ES-PLUGIN](https://s12.postimg.org/53rhsd56l/kibana.png "KIBANA")
#
![AWS-ES-PLUGIN](https://s15.postimg.org/aep66lyyz/es_plugin_head.png "AWS-ES-PLUGIN_HEAD")
#
![AWS-ES-PLUGIN](https://s27.postimg.org/6dulkv2z7/es_plugin_bigdesk.png "AWS-ES-PLUGIN_BIGDESK")
#
![AWS-ES-PLUGIN](https://s10.postimg.org/5bacv91rt/bigdesk_cluster.png "AWS-ES-PLUGIN_BIGDESK")
#

