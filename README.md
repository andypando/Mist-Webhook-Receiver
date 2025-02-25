# Mist-Webhook-Receiver

This project was created to give you the steps to setup an Ubuntu server and install the ELK stack on it.
Logstash will be configured as an HTTP Receiver to accept the webhooks from Mist, they will be 
processed into Elasticsearch and Kibana allows users to search the webhooks and build useful dashboards
with the data.

Dashboard JSON files can be committed as folks build useful things in their environment. The hope is that this
will be a collaborative project that will be useful to anyone using Juniper Mist APs.


# Installation Steps


## Configure Router / Firewall for inbound webhooks
All the Mist webhooks originate from the Mist cloud, so you will need inbound firewall and/or NAT rules to direct
these webhooks to your ELK server.

| Mist Cloud  | Webhook IP Address |
| ------------- | ------------- |
| Global 01 - manage.mist.com  | 54.193.71.17 & 54.215.237.20	  |
| Global 03 - manage.ac2.mist.com | 34.231.34.177 & 54.235.187.11 & 18.233.33.230  |

For this example we are using port 8899, but you can choose any port number you like just be sure to change
the port number to the one you are using throught this walk through.

You will need an inbound / destination NAT rule for the webhooks and it can further be limited to the source
IP addresses for the cloud instance your Org. resides in, see table above.

This is an example of the NAT rules configured on a Mist WAN Edge device:

![Image Alt](https://github.com/andypando/Mist-Webhook-Receiver/blob/785d36ce973b9e7678698ecb6b55fb58739a0073/NAT.png)


## Setup Ubuntu Server

You will need an Ubuntu server running to host the services required. This could be either a Virtualized Host or physical box.
The compute required is minimal, but the storage will dictate how many webhook topic you can send as well as the retention time
of the raw data. This walk through will use the current Ubuntu 24.04.2 LTS image, available for download here: [Ubuntu Server Download](https://ubuntu.com/download/server "Ubuntu Server Download")
While not a neccessity, you could use a version of Ubuntu that includes a desktop environment, but all instructions will be based off of a server version and will have CLI commands.

Here is the VM Ware settings for a basic, minimal server to play with. Note that the HDD is thin provisioned, this allows for more dynamic sizing.
For a production machine, it would be recommended to increase the HDD size.

![Image Alt](https://github.com/andypando/Mist-Webhook-Receiver/blob/a68a0998978eb980c64f30b3f03f5d19742e41c0/Sample_VM.png)

Continue through setup, if using a server image you can install minimal packages. At Network Configuration, 
you will want to change from DHCP to Manual and setup a static IP for this server to use. It will be the same 
server address used on the inside of your NAT rules. If asked, install SSH server. It is quicker to SSH into this
server and copy/paste the commands from here.

### Make yourself root
SSH to the server using the username/password that you setup on install. Switch to a root user:
```
sudo su -
```

### Install OpenJDK 11 & curl
```
apt update && apt upgrade -y && apt install -y default-jre default-jdk curl
```

 ### Install Elasticsearch
 ```
curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elastic.gpg
```
