## Discovery Catalogue deployment

### Overview

This page describes the deployment of the WIS2 GDC on the MET5 infrastructure testing environment, available at https://met5node.weather.gc.ca.

#### Infrastructure

The WIS2 GDC is deployed to Amazon Web Services (EC2 t2.medium with 24GB storage), with ports 443 and 8883 enabled.

#### Software

The WIS2 GDC is powered by the [wis2-gdc](https://github.com/wmo-im/wis2-gdc) Reference Implementation.

### Deployment

#### Initial setup

```bash
sudo apt update
sudo apt install make git vim unzip nginx  -y
# install Docker as per https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository
# update Docker group privileges as per https://docs.docker.com/engine/install/linux-postinstall 
```

#### Certificates

Certificates are managed and provided by Shared Service.

#### Nginx

Nginx is installed on the host system with the following updates to `/etc/nginx/sites-available/met5node.weather.gc.ca`:

```
server {
        listen 443 ssl default_server;
        listen [::]:443 ssl default_server;

        ssl_certificate /etc/letsencrypt/live/met5node.weather.gc.ca/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/met5node.weather.gc.ca/privkey.pem;

        root /var/www/html;

        server_name met5node.weather.gc.ca;

        location / {
                proxy_pass http://localhost:5001/;
        }

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Host $server_name;

        add_header 'Access-Control-Allow-Origin' '*';
        add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';
        add_header 'Access-Control-Allow-Headers' 'DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type';
}
```

Streaming MQTT over port 8883 requires the following updates in `/etc/nginx/nginx.conf`:


```
stream {
        upstream broker {
                server 127.0.0.1:1883 fail_timeout=1s max_fails=1;
        }
        server {
                ssl_certificate /etc/letsencrypt/live/met5node.weather.gc.ca/fullchain.pem;
                ssl_certificate_key /etc/letsencrypt/live/met5node.weather.gc.ca/privkey.pem;
                ssl_protocols TLSv1.2;
                listen 8883 ssl;
                proxy_pass broker;
                proxy_connect_timeout 1s;
        }
}
```

```bash
sudo service nginx stop
sudo ln -s /etc/nginx/sites-available/met5node.weather.gc.ca /etc/nginx/sites-enabled/met5node.weather.gc.ca
sudo service nginx start
```

#### wis2-gdc

The WIS2 GDC is deployed as follows:

```bash
git clone https://github.com/wmo-im/wis2-gdc.git
cd wis2-gdc
```

The default setup requires the following changes:

```diff
diff --git a/docker-compose.override.yml b/docker-compose.override.yml
index 79d4c79..29ddb23 100644
--- a/docker-compose.override.yml
+++ b/docker-compose.override.yml
@@ -26,7 +26,7 @@ services:
       - 1884:1884  # websockets
   wis2-gdc-api:
     ports:
-      - 80:80
+      - 5001:80
   grafana:
     ports:
       - 3000:3000
diff --git a/wis2-gdc.env b/wis2-gdc.env
index a4cd1db..205b354 100644
--- a/wis2-gdc.env
+++ b/wis2-gdc.env
@@ -1,20 +1,17 @@
 export WIS2_GDC_LOGGING_LEVEL=DEBUG
-export WIS2_GDC_API_URL=http://localhost
+export WIS2_GDC_API_URL=https://met5node.weather.gc.ca
 export WIS2_GDC_API_URL_DOCKER=http://wis2-gdc-api
 export WIS2_GDC_BACKEND_TYPE=Elasticsearch
 export WIS2_GDC_BACKEND_CONNECTION=http://wis2-gdc-backend:9200/wis2-discovery-metadata
 export WIS2_GDC_BROKER_URL=mqtt://wis2-gdc:wis2-gdc@wis2-gdc-broker:1883
-export WIS2_GDC_CENTRE_ID=ca-eccc-msc-global-discovery-catalogue
+export WIS2_GDC_CENTRE_ID=ca-eccc-msc
 export WIS2_GDC_COLLECTOR_URL=http://wis2-gdc-metrics-collector:8006
-export WIS2_GDC_GB=mqtts://everyone:everyone@wis2globalbroker.nws.noaa.gov:8883
+export WIS2_GDC_GB=${WIS2_GDC_BROKER_URL}
 export WIS2_GDC_GB_TOPIC=cache/a/wis2/+/metadata/#
 export WIS2_GDC_METADATA_ARCHIVE_ZIPFILE=/data/wis2-gdc-archive.zip
 export WIS2_GDC_PUBLISH_REPORTS=true
 export WIS2_GDC_REJECT_ON_FAILING_ETS=false
 export WIS2_GDC_RUN_KPI=true
 
-# global broker links
-export WIS2_GDC_GB_LINK_METEOFRANCE="fr-meteo-france-global-broker,mqtts://everyone:everyone@globalbroker.meteo.fr:8883,Météo-France, Global Broker Service"
-export WIS2_GDC_GB_LINK_CMA="cn-cma-global-broker,mqtts://everyone:everyone@gb.wis.cma.cn:8883,China Meteorological Agency, Global Broker Service"
-export WIS2_GDC_GB_LINK_NOAA="us-noaa-nws-global-broker,mqtts://everyone:everyone@wis2globalbroker.nws.noaa.gov:8883,National Oceanic and Atmospheric Administration, National Weather Service, Global Broker Service"
-export WIS2_GDC_GB_LINK_INMET="br-inmet-global-broker,mqtts://everyone:everyone@globalbroker.inmet.gov.br:8883,Instituto Nacional de Meteorologia (Brazil), Global Broker Service"
+# broker links
+export WIS2_GDC_GB_LINK_METEOFRANCE="ca-eccc-msc,${WIS2_GDC_BROKER_URL},Environment and Climate Change Canada, Meteorological Service of Canada, Global Broker Service"
```

At this point the wis2-gdc can be started as follows by running the `Makefile` startup target:

```bash
make up
```

The wis2-gdc should then be available at https://met5node.weather.gc.ca
