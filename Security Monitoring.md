# Security Monitoring Overview

As a security team, we need to make sure that both the VMWare servers (aka ESXI) and the virtual machines deployed in it, are monitored. 

From Linux standpoint, the auth logs provides a good visibility on the authentication attempts on those servers. When adding a Network Intrussion Prevension platform (IPS) such as Fail2ban it is important to collect and centralize both events.

Adding a IPS to our current deployement becomes a security layered approach because we are mitigating the event of leaving a server misconfigured.

# Steps

1. Configure FluentBit to collect auth.logs and Fail2Ban logs: The following configuration defines two entry points of collection.

Fluent Bit configuration file is located at ```/etc/td-agent-bit/td-agent-bit.conf```

In this case, we are defining two Input Sections and the Output is sent to Logstash. using the HTTP Protocol, the HTTP Port 12345 and the IP Address of the Logstash Server.	

```
[INPUT]
    name tail
    tag  fail2ban.file
    Path /var/log/fail2ban.log
    DB   /var/log/fail2ban_error.db
    Read_from_head true
    Mem_Buf_Limit     8MB
    Skip_Long_Lines   On
    Refresh_Interval  30
[INPUT]
    name tail
    tag  auth.file
    Path /var/log/auth.log
    DB   /var/log/auth_error.db
    Read_from_head true
    Mem_Buf_Limit     8MB
    Skip_Long_Lines   On
    Refresh_Interval  30
[FILTER]
    Name record_modifier
    Match *
    Record hostname ${HOSTNAME}
[OUTPUT]
    name  http
    match *
    host 0.0.0.0
    port 12345
    format json

```

**Be mindful that the auth log location is slighly different for ESXI servers (/var/run/auth.log)**

```
[INPUT]
    name tail
    tag  auth.file
    Path /var/run/log/auth.log
    DB   /var/run/log/auth_error.db
    Read_from_head true
    Mem_Buf_Limit     8MB
    Skip_Long_Lines   On
    Refresh_Interval  30
```

# Logstash Configuration

For this setup, thee Logstash configuration files have been created, they need to be located at /etc/logstash/config.d in the Logstash server:

* 001-input.conf
* 002-filter.conf
* 003-output.conf

Input Configuration file looks like this:

```

input {
  http {
    host => "0.0.0.0" 
    port => 12345
  }
}

``` 

**Note that this is an HTTP input and the port number must match the port number defined in the Fluent Bit configuration file **

We need to copy the attached Filebeat configuration files in the Logstash server.

# Logstash output configuration

Following Logstash output configuration creates an index per month. This value is prefered, however it can be changed:

`output {
      elasticsearch {
	hosts => ["localhost:9200"]
        index => "linux-auth-%{+YYYY.MM}"
	}
 }`
 

# Logstash Grok Parsers

For Parsing auth.log and Fail2ban.log file, some grok parser files are required. In the logstash server, it is necesary to create a patterns folder and place the attached grok files.

After this step, the patterns folder should look like this:


```
ls -l /etc/logstash/patterns/
total 24
-rw-r--r-- 1 root root  172 Jan  8 14:52 fail2ban-log-patterns.grok
-rw-r--r-- 1 root root  531 Jan  8 18:34 pam.grok
-rw-r--r-- 1 root root  681 Jan  8 18:34 sshd.grok
-rw-r--r-- 1 root root  223 Jan  8 18:34 sudo.grok
-rw-r--r-- 1 root root  234 Jan  8 18:34 systemd.grok
-rw-r--r-- 1 root root 1004 Jan  8 18:34 user-management.grok
root@mike-server:~# 

```

# Starting Logstash 

Make sure that Logstash service runs at the system startup (this is important in the event the system reboot)

```
systemctl enable logstash.service

```


# Elasticsearch Index Mapping

Linux authentication logs and Fail2Ban will contain the remote client IP address. This field is extracted and enriched with geolocation. However, to make it a geo point to be displayed in map, we need to define an index mapping running the following command in elasticsearch:

```

PUT _index_template/linux-auth
{
	  "index_patterns": ["*auth*"],
	  "template": {
	    "mappings": {
		      "properties": {
		        "geoip"  : {
		          "dynamic": true,
		          "properties" : {
		            "ip": { "type": "ip" },
		            "location" : { "type" : "geo_point" },
		            "latitude" : { "type" : "half_float" },
		            "longitude" : { "type" : "half_float" }
		          }
		        }
		      }
	  		}
	  }
}
```

# Importing Kibana artifacts

Some visualizations, index patterns and dashboards have been created, however those can be imported once data has been loaded. Please use the attached bundle to upload the artifacts to Elasticsearch.

Go to Kibana -> Stack Management -> Saved Objects -> Import

