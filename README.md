# nginxlb-docker

A really simple HTTP load balancer/router designed to run on CoreOS, uses etcd as a datastore

## Quick Start

Start an instance of the load balancer on every node of your cluster

```[Unit]
Description=Nginx FELB
After=docker.service
After=skydns.service

[Service]
User=core
TimeoutStartSec=0
EnvironmentFile=/etc/environment
ExecStartPre=-/usr/bin/docker kill nginx-lb
ExecStartPre=-/usr/bin/docker rm nginx-lb
ExecStartPre=/usr/bin/docker pull insertjokehere/nginxlb
ExecStart=/usr/bin/docker run --rm --name nginx-lb -p 80:80 --dns=${COREOS_PRIVATE_IPV4} -e ETCD_IP=${COREOS_PRIVATE_IPV4} insertjokehere/nginxlb
ExecStop=/usr/bin/docker kill nginx-lb

[X-Fleet]
Global=true```

Then create a sidekick unit for each service that needs connections routed to it. For example, to serve api.testapi.skydns.local:5000 as http://api.example.com/

```[Unit]
Description=api service register
BindsTo=api.service
After=api.service

[Service]
RemainAfterExit=yes
ExecStart=/bin/sh -c "while true; do /bin/etcdctl set \"/lb/http/api.example.com/upstreams/root/path\" / --ttl 60; \
 /bin/etcdctl set \"/lb/http/api.example.com/upstreams/root/backends/api\" api.testapi.skydns.local:5000 --ttl 60; sleep 30; done"

[X-Fleet]
X-ConditionMachineOf=api.service```
