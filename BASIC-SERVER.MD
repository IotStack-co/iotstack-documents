# How to install a production-ready IIoT server (basic manual setup)

## Introduction and Prerequisites

This document shows the fresh manual installation of an IIoT stack on Ubuntu 18.04 using well-known 
open source components including InfluxDB, Telegraf, Grafana (i.e. the TIG-Stack) and Mosquitto as an MQTT-Broker for connectivity of
devices. 

## Setting up Let's Encrypt
As a prerequisite, a domain or sub-domain must be created with a provider and referenced to the IP of the server via A-Record.

Certbot PPA has to be added to the list of repositories.

```
sudo apt-get update
sudo apt-get install software-properties-common
sudo add-apt-repository universe
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get update
```

Install Certbot

```
sudo apt-get install certbot
```

Run this command to get a certificate. Certbot will temporarily spin up a webserver

```
sudo certbot certonly --standalone -d {your_domain}
```

The certificates are then located in: `/etc/letsencrypt/live/{your_domain}/`


## Installation of InfluxDB

Add the InfluxData repository with the following commands:

```
wget -qO- https://repos.influxdata.com/influxdb.key | sudo apt-key add -
source /etc/lsb-release
echo "deb https://repos.influxdata.com/${DISTRIB_ID,,} ${DISTRIB_CODENAME} stable" | sudo tee /etc/ap
```

Install and start the InfluxDB service:

```
sudo apt-get update && sudo apt-get install influxdb
sudo service influxdb start
```

To secure Influx with SSL, change `/etc/influxdb/influxdb.conf`. 

```
# Determines whether HTTPS is enabled.
  https-enabled = true
 
  # The SSL certificate to use when HTTPS is enabled.
  https-certificate = "/etc/letsencrypt/live/{your_domain}/fullchain.pem"
 
  # Use a separate private key location.
  https-private-key = "/etc/letsencrypt/live/{your_domain}/privkey.pem"
```

## Installation of Grafana

Add Repository and install

```
sudo apt-get install -y apt-transport-https
sudo apt-get install -y software-properties-common wget
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
sudo apt-get update
sudo apt-get install grafana
```

Then start

```
sudo service grafana-server start
```
```
setcap 'cap_net_bind_service=+ep' /usr/sbin/grafana-server 
```
should allow the service to bind to privileged ports in case of any problems with the startup.

### Edit Grafana.ini 
Config File of Grafana should be edited to support https:

```
[server]
# Protocol (http, https, socket)
protocol = https
 
# The http port to use
http_port = 443
 
# The public facing domain name used to access grafana from a browser
domain = your.grafana.url
 
# The full public facing url you use in browser, used for redirects and emails
# If you use reverse proxy and sub path specify full url (with sub path)
root_url = https://your.grafana.url
 
# https certs & key file
cert_file = /etc/letsencrypt/live/{your_domain}/fullchain.pem
cert_key = /etc/letsencrypt/live/{your_domain}/privkey.pem
```

## Installation of node.js
```
sudo apt update
sudo apt install nodejs
sudo apt install npm
nodejs -v
```

## Installation os Mosquitto

```
sudo apt-get update
sudo apt-get install mosquitto
```

Then configuration adjustments must be made:

### Creating users

A user in Mosquitto stands for a device that connects to the broker to exchange data. A user must also be created 
for the telegraph service, which writes the data of the devices into the Influx.

```
sudo mosquitto_passwd -c /etc/mosquitto/passwd <your-username>
```

### Configuration

Create File with

```
sudo nano /etc/mosquitto/conf.d/default.conf
```

Then insert the following configuration:

```
allow_anonymous false
password_file /etc/mosquitto/passwd
 
listener 1883 localhost
 
listener 8883
certfile /etc/letsencrypt/live/{your_domain}/cert.pem
cafile /etc/letsencrypt/live/{your_domain}/chain.pem
keyfile /etc/letsencrypt/live/{your_domain}/privkey.pem
 
listener 8083
protocol websockets
certfile /etc/letsencrypt/live/{your_domain}/cert.pem
cafile /etc/letsencrypt/live/{your_domain}/chain.pem
keyfile /etc/letsencrypt/live/{your_domain}/privkey.pem
```

## Firewall UFW

Then open the ports of the firewall for all required systems

```
sudo ufw allow 8883
sudo ufw allow 3000
sudo ufw allow 8083
sudo ufw allow ssh
sudo ufw enable
sudo systemctl restart mosquitto
```

## Installation of Telegraf

To install telegraf use

```
cat <<EOF | sudo tee /etc/apt/sources.list.d/influxdata.list
deb https://repos.influxdata.com/ubuntu bionic stable
EOF
 
 
sudo apt-get update
sudo apt-get install telegraf
```

