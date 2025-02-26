# Mist-Webhook-Receiver

This project was created to give you the steps to setup an Ubuntu server and install the [ELK stack](https://www.elastic.co/) on it.
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

### Install OpenJDK 11 & other Utils

```
apt update && apt upgrade -y && apt install -y default-jre default-jdk curl tcpdump
```

 ### Install Elasticsearch

 ```
curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elastic.gpg
```

```
echo "deb [signed-by=/usr/share/keyrings/elastic.gpg] https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list
```

```
apt update && apt install -y elasticsearch
```

### Configure Elasticsearch

```
cp /etc/elasticsearch/elasticsearch.yml /etc/elasticsearch/elasticsearch.bak
```

```
apt install nano && nano /etc/elasticsearch/elasticsearch.yml
```

**Under Network, set the following for network.host, http.host, http.port**

![Image Alt](https://github.com/andypando/Mist-Webhook-Receiver/blob/be6a8b58ffa6b267808cd05f339124ef9465cbb0/ES_Config.png)

**Under Security add**

```
xpack.security.enabled: true
```

![Image Alt](https://github.com/andypando/Mist-Webhook-Receiver/blob/c3f2f5442990b39911833dea4d9d1eab919e3dc5/ES_Security.png)

For new users, Ctrl+x will exit you from nano text editor and prompt you to save changes.

### Start Elasticsearch

```
systemctl start elasticsearch
```

### Setup Elasticsearch passwords - These can call be the same or different, but make it something other than user password

```
/usr/share/elasticsearch/bin/elasticsearch-setup-passwords interactive
```

### Set Elasticsearch to run at boot

```
systemctl enable elasticsearch
```

### Install and enable Kibana

```
apt install kibana
```

### Comment out legacy OpenSSL to disable it

```
nano /etc/kibana/node.options
```

![Image Alt](https://github.com/andypando/Mist-Webhook-Receiver/blob/09529ad5f50884154336045a3f4ccf98b7c3d7c0/Kibana_Sec.png)

### Configure Kibana

```
cp /etc/kibana/kibana.yml /etc/kibana/kibana.bak
```

```
nano /etc/kibana/kibana.yml
```

**Set IP to server IP address and Port to 5601**

![Image Alt](https://github.com/andypando/Mist-Webhook-Receiver/blob/9d9fad594c5972321999840dad36a71c45b34e2e/Kibana_1.png)

**Set uncomment Elasticsearch Username and Password, be sure PW is what you set during Elasticsearch Setup**

![Image Alt](https://github.com/andypando/Mist-Webhook-Receiver/blob/9d9fad594c5972321999840dad36a71c45b34e2e/Kibana_2.png)

### Enable Kibana at boot and restart service

```
systemctl enable kibana && systemctl restart kibana
```

### Install Logstash

```
apt install logstash
```

### Setup http listener

```
nano /etc/logstash/conf.d/http.conf
```

**Paste in the following code block, *BE SURE TO CHANGE PASSWORD***

```
input {
  http {
    host => "0.0.0.0"
    port => 8899
  }
}
filter {
  split { field => "[events]" }
  if [topic] == "client-join" {
    mutate {add_field => {"radius_acct" => "no"}}
  }
}
output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "%{[topic]}-%{+YYYY.MM.dd}"
    user => "elastic"
    password => "--Password set above--"
  }
}
```

### Enable Logstash at boot and start service
```
systemctl start logstash && systemctl enable logstash
```

## Configure Webhook in Mist

This can be enabled for either the entire Org or for individual site:

Entire Org: Organization -> Settings
Specific Sites: Organization -> Site Configuration -> Click Into the Site

Enable client-join and any other webhooks you want to send. Use your outside NAT IP and port 8899

![Image Alt](https://github.com/andypando/Mist-Webhook-Receiver/blob/21cb6a0d556b2a3991ff2867ebd8829bedcda77f/Mist_Web.png)

## Test webhooks

Find the interface in use by your server, use:

```
ip a
```

This will list all the active interfaces, find the one that has your static IP address. It will likely be eno1 or ens160

Run one of the following commands (depending on the interface or edit as needed) on the server to see if webhook are getting forwarded to it properly. You may have to bounce some clients to trigger new join webhook to get fired.

```
tcpdump -Xni eno1 port 8899
```
_or_
```
tcpdump -Xni ens160 port 8899
```

You should see something similar to:

> 01:36:31.872892 IP 54.215.237.20.3854 > 10.10.10.12.8899: Flags [.], ack 2201270769, win 491, options [nop,nop,TS val 2775259957 ecr 740392647], length 0  
>	0x0000:  4500 0034 df59 4000 3606 2d69 36d7 ed14  E..4.Y@.6.-i6...  
>	0x0010:  0a0a 0a0c 0f0e 22c3 bf74 f896 8334 b9f1  ......"..t...4..  
>	0x0020:  8010 01eb aa44 0000 0101 080a a56b 1b35  .....D.......k.5  
>	0x0030:  2c21 7ec7  

To exit out of tcpdump, press Ctrl-c

## Finish setting up Kibana

* Using a browser on a computer that can access your server, navigate to to it on port 5601 -> http://<server_IP>:5601
* Username will be _elastic_ and the password will be what you set when configuring Elasticsearch
* If you are receiving webhooks and Logstash if forwarding them properly, you may be prompted to setup Indexes when you log in, or go to Management > Stack Management > Index Patterns
* Setup an Index Pattern for each webhook coming in, use @timestamp for Timestamp Field

![Image Alt](https://github.com/andypando/Mist-Webhook-Receiver/blob/5555bc1ab8e95d5371a52cede03b8a56515939bb/Index.png)
