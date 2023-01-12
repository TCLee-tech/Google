# Set Up Network and HTTP Load Balancers

To learn:
How to set up a [network load balancer](https://cloud.google.com/load-balancing/docs/network) with target pools backend (outdated legacy).
How to set up an [external HTTP(S) load balancer](https://cloud.google.com/load-balancing/docs/https) with managed instance group backend.

[Cloud Load Balancing Overview](https://cloud.google.com/load-balancing/docs/load-balancing-overview)
[Difference between Layer 4 vs Layer 7 Load Balancing](https://medium.com/@harishramkumar/difference-between-layer-4-vs-layer-7-load-balancing-57464e29ed9f)
[NGINX Difference between Layer 4 and 7 Load Balancing](https://www.nginx.com/resources/glossary/layer-7-load-balancing/)


### Task 1. Set the default region and zone for all resources
[Regions and Zones Guide](https://cloud.google.com/compute/docs/regions-zones)  
`gcloud config set compute/zone`
`gcloud config set compute/region`

### Task 2. Create multiple web server instances
1. For lab scenario, create 3 VM instances
  - install Apache
  - tag the VMs for easy reference, e.g. during creation of firewall rules.
  ```
    gcloud compute instances create www1 \  <= change name when repeating commands for www2, www3
    --zone= \
    --tags=network-lb-tag \
    --machine-type=e2-small \
    --image-family=debian-11 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
      apt-get update
      apt-get install apache2 -y
      service apache2 restart
      echo "
<h3>Web Server: www1</h3>" | tee /var/www/html/index.html'
```

2. Create firewall rules to allow external traffic to reach VMs  
`gcloud compute firewall-rules create www-firewall-network-lb --target-tags network-lb-tag --allow tcp:80`
3. To verify VMs running and can be reached from outside Google Cloud  
list instances `gcloud compute instances list`  
transmit data over url using Client URL `curl http://[IP_ADDRESS]`  

### Task 3. Configure the L4 network load balancing service
1. Create static external IP address
2. Add legacy HTTP health check resource
3. Create target pool in same region as VM instances
4. Add instances to target pool
5. Add forwarding rule to forward traffic from external IP address to target pool

`gcloud compute addresses create network-lb-ip-1 --region`
`gcloud compute http-health-checks create basic-check`
`gcloud compute target-pools create www-pool --region --http-health-check basic-check`
`gcloud compute target-pools add-instances www-pool --instances www1,www2,www3`
`gcloud compute forwarding-rules create www-rule --region --ports 80 --address network-lb-ip-1 --target-pool www-pool`

### Task 4. Sending traffic to your instances
To verify. Observe that traffic sent to external IP address forwarded to different backend instances.
1. View external IP address of forwarding rule
`gcloud compute forwarding-rules describe www-rule --region`
2. Access the IP address
`IPADDRESS=$(gcloud compute forwarding-rules describe www-rule --region  --format="json" | jq -r .IPAddress)`
3. Show the IP address
`echo $IPADDRESS`
4. Send data over url to the IP addresses
`while true; do curl -m1 $IPADDRESS; done`
`CTRL + c` to stop running command.
