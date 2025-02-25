# Mist-Webhook-Receiver

This project was created to give you the steps to setup an Ubuntu server and install the ELK stack on it.
Logstash is will be configured as a HTTP Receiver to accept the webhooks from Mist, they will be 
processed into Elasticsearch and Kibana allows users to search the webhooks and build useful dashboards
with the data.

Dashboard JSON files can be committed as folks build useful things in their environment. The hope is that this
will be a collaborative project that will be useful to anyone using Juniper Mist APs.

# Allow for inbound webhooks

All the Mist webhooks originate from the Mist cloud, so you will need inbound firewall NAT rules to direct
these webhooks to your ELK server.

| Mist Cloud  | Webhook IP Address |
| ------------- | ------------- |
| Global 01 - manage.mist.com  | 54.193.71.17 & 54.215.237.20	  |
| Global 03 - manage.ac2.mist.com | 34.231.34.177 & 54.235.187.11 & 18.233.33.230  |

For this example we are using port 8899, but you can choose any port number you like just be sure to change
the port number to the one you are using throught this walk through.

You will need an inbound / destination NAT rule for the webhooks and it can further be limited to the source
IP addresses for the cloud instance your Org. resides in, see table above.

