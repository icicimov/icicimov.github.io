---
layout: post
comments: true
author:
  name: Igor Cicimov
  email: igorc@encompasscorporation.com
description: 'Dynamically adjust server weight using external agent in HAProxy'
keywords: 'haproxy'
license: '[CC BY-ND 4.0](https://creativecommons.org/licenses/by-nd/4.0)'
title: HAProxy dynamically adjust server weight using external agent
category: Server
tags: haproxy
---

Trying to utilize HAProxy-1.5/1.6 `agent-check` feature, see [HAProxy documentation](http://cbonte.github.io/haproxy-dconv/configuration-1.5.html#5.2-agent-check), I wrote this small script to check Tomcat system load and return back some values that HAP can use to dynamically adjust the server weight in the backend.

This will run as xinetd service on the Tomcats, for example made available to HAP on port 9707 (some randomly chosen free port).

Explanation how is this going to work. Each server starts with weight of 100. This health check will run every 5 minutes lets say (the primary runs every 10 seconds) and the agent will return "up" plus weight value or "down" upon heath check. The weight percentage returned is calculated based on the system load in the last 5 minutes (can be 1 or 15 min. too), ie if load is below 90% it will return weight of 100, between 90% and 100% it will reduce the weight to 50% (meaning HAP will send only 25% of the connections to this server in case of 2 servers) and if 100% it will mark this server down.

This additional, lets call it secondary, heath check will work in conjunction with the already existing one, which only checks if the service is up or down, and provide flexibility in terms of backend load.

## Setup

### On the backends (Tomcat app servers)

Create agent-check script `/usr/local/bin/haproxy-agent-check`:

```bash
#!/bin/bash
LMAX=90
 
load=$(uptime | grep -E -o 'load average[s:][: ].*' | sed 's/,//g' | cut -d' ' -f3-5)
cpus=$(grep processor /proc/cpuinfo | wc -l)
 
while read -r l1 l5 l15; do {
    l5util=$(echo "$l5/$cpus*100" | bc -l | cut -d"." -f1);
    [[ $l5util -lt $LMAX ]] && echo "up 100%" && exit 0;
    [[ $l5util -gt $LMAX ]] && [[ $l5util -lt 100 ]] && echo "up 50%" && exit 0;
    echo "down#CPU overload";
}; done < <(echo $load)
 
exit 0
```

Set it to be executable:

```
root@app21:~# chmod +x /usr/local/bin/haproxy-agent-check
```

Next we set it as xinetd service, first install xinetd:

```
root@app21:~# aptitude install -y xinetd
```

Then add our service to the system services in `/etc/services` file:

```
...
haproxy-agent-check 9707/tcp                # haproxy-agent-check
...
```

Create the xinetd daemon file `/etc/xinetd.d/haproxy-agent-check`:

```
# default: on
# description: haproxy-agent-check
service haproxy-agent-check
{
        disable         = no
        flags           = REUSE
        socket_type     = stream
        port            = 9707
        wait            = no
        user            = nobody
        server          = /usr/local/bin/haproxy-agent-check
        log_on_failure  += USERID
        only_from       = 172.31.17.11 172.31.11.11 127.0.0.1
        per_source      = UNLIMITED
}
```

To allow access from specific subnets instead of hosts we can use the following format:

```
only_from       = 10.22.0.0 10.22.1.0 10.22.2.0 127.0.0.1
```

Make it executable, restart xinetd and test:

```
root@app21-hap:~# chmod +x /etc/xinetd.d/haproxy-agent-check
root@app21-:~# service xinetd restart
root@app21:~# telnet 127.0.0.1 9707
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
up 100%
Connection closed by foreign host.
```

All looks good so we can setup the HAP's now.

# On the HAProxy LB's

Open the TCP port 9707 in the APP servers firewall and Test the agent-check connectivity from HAP serers to the app servers:

```
root@lb1:~# telnet ip-172-31-17-41 9707
Trying 172.31.17.41...
Connected to ip-172-31-17-41.ap-southeast-2.compute.internal.
Escape character is '^]'.
up 100%
Connection closed by foreign host.
 
root@lb1:~# telnet ip-172-31-11-41 9707
Trying 172.31.11.41...
Connected to ip-172-31-11-41.ap-southeast-2.compute.internal.
Escape character is '^]'.
up 100%
Connection closed by foreign host.
root@lb1:~#
```

Add `agent-check`, `agent-port` and `agent-inter` parameters to the backend servers in the `/etc/haproxy/haproxy.cfg` config file:

```
...
listen https-in
...
    default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 200 maxqueue 250 weight 100 error-limit 10 on-error mark-down on-marked-down shutdown-sessions agent-port 9707 agent-inter 30s
    server ip-172-31-17-41  ip-172-31-17-41:8080  check agent-check observe layer7
    server ip-172-31-11-41  ip-172-31-11-41:8080  check agent-check observe layer7
    server localhost 127.0.0.1:8080 maxconn 500 backup weight 1
```

And reload HAP service. Check the health status:

```
root@lb1:~# echo "show stat" | socat stdio unix-connect:/run/haproxy/admin.sock | cut -d ',' -f1,2,18,19 | grep https
https-in,ip-172-31-17-41,UP,100
https-in,ip-172-31-11-41,UP,100
https-in,localhost,no check,1
https-in,BACKEND,UP,200
```

We can see the weight of both app servers is set to 100 atm but this should change if/when they come under high load for 5 minutes.
