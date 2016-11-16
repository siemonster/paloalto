# paloalto
#SIEMonster paloalto integration
### Based on PAN OS 7.1.0

PAN NGFW - Device - Syslog
Create 2 profiles, 1 for traffic, one for threats (URLs).

![pa-device](https://cloud.githubusercontent.com/assets/16313160/17761940/daf15704-654e-11e6-92ad-3678de208715.png)

Set the IP of the Syslog Server (external IP of Proteus). Set the port and protocol, in this example 3526 TCP.
Use the LOG-LOCAL0 facility for traffic and LOG-LOCAL1 for threats.

![pa-log0](https://cloud.githubusercontent.com/assets/16313160/17761937/daeca092-654e-11e6-8d32-8625c56b93f8.png)

![pa-log1](https://cloud.githubusercontent.com/assets/16313160/17761941/daf2203a-654e-11e6-9332-8d1ae6c52013.png)

Go to the Objects tab, then Log Forwarding. Create a new profile, turn on syslogs for Traffic (the any severity) and for Threats (the information setting).

![pa-objects](https://cloud.githubusercontent.com/assets/16313160/17761939/daefdc1c-654e-11e6-8d39-5e36331d57e8.png)

On the Policies tab add the log forwarding profile to the desired policy.

![pa-policy](https://cloud.githubusercontent.com/assets/16313160/17761938/daeca86c-654e-11e6-9b0f-1cb9256457a5.png)

Add a custom policy to the URL Syslog, using the fields shown in Threat item.

![pa-custom](https://cloud.githubusercontent.com/assets/16313160/17762730/93f5dc64-6556-11e6-8c55-079c27abc38d.png)

On the Syslog Server (Proteus), configure appropriate source, destinations, and filters.
Edit /etc/syslog-ng/syslog-ng.conf and incorporate the following changes.

```
source s_netsyslog {
       tcp(ip(0.0.0.0) port(514));
       tcp(ip(0.0.0.0) port(3526));
       udp(ip(0.0.0.0) port(514));
       udp(ip(0.0.0.0) port(1514));
};

destination d_netsyslog { file("/var/log/traffic.log" owner("logstash") group("root") perm(0644)); };
destination d_urlsyslog { file("/var/log/urllogs.log" owner("logstash") group("root") perm(0644)); };

filter f_traffic { facility(local0); };
filter f_threat { facility(local1); };

log { source(s_netsyslog); filter(f_traffic); destination(d_netsyslog); };
log { source(s_netsyslog); filter(f_threat); destination(d_urlsyslog); };
```
Prepare the Elasticsearch mapping:
```
curl -XPUT localhost:9200/_template/pan-traffic -d@pan-traffic-mappings.json
curl -XPUT localhost:9200/_template/pan-url -d@pan-url-mappings.json
```
Logstash inputs can be configured as follows:

```
input {
file {
        path => ["/var/log/traffic.log"]
        type => "traffic"
        tags => ["paloalto"]
        }
 file {
        path => ["/var/log/urllogs.log"]
        type => "url"
        tags => ["paloalto"]
        }
}
```
The logstash filter 25-paloalto-filter.conf can be downloaded from this repository.

Logstash outputs can be configured as follows.

```
output {
if [type] == "traffic" {
     elasticsearch {
         hosts => ["localhost:9200"]
         index => "pan-traffic-%{+YYYY.MM.dd}"
          }
    }
else if [type] == "url" {
    elasticsearch {
         hosts => ["localhost:9200"]
         index => "pan-url-%{+YYYY.MM.dd}"

          }
       }

```

Register each index in Kibana, pan-traffic-* & pan-url-*

![pa-index](https://cloud.githubusercontent.com/assets/16313160/17763077/877b29dc-6559-11e6-8578-f2701cb63511.png)
