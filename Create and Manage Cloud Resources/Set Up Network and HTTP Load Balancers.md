# Set Up Network and HTTP Load Balancers

Learning:  
How to set up a [network load balancer](https://cloud.google.com/load-balancing/docs/network) with target pools backend. [outdated legacy]  
How to set up an [external HTTP(S) load balancer](https://cloud.google.com/load-balancing/docs/https) with managed instance group backend.

Reading material:  
[Cloud Load Balancing Overview](https://cloud.google.com/load-balancing/docs/load-balancing-overview)  
[Difference between Layer 4 vs Layer 7 Load Balancing](https://medium.com/@harishramkumar/difference-between-layer-4-vs-layer-7-load-balancing-57464e29ed9f)  
[NGINX Difference between Layer 4 and 7 Load Balancing](https://www.nginx.com/resources/glossary/layer-7-load-balancing/)  

<hr>

### Task 1. Set the default region and zone for all resources
[Regions and Zones Guide](https://cloud.google.com/compute/docs/regions-zones)  
`gcloud config set compute/zone`  
`gcloud config set compute/region`  

### Task 2. Create multiple web server instances
1. For this lab scenario, create 3 VM instances
  - install Apache
  - tag the VMs for easy reference during creation of firewall rules
  ```
    gcloud compute instances create www1 \  <= change name when repeating commands for www2, www3
    --zone= \
    --tags=network-lb-tag \     <= tag
    --machine-type=e2-small \
    --image-family=debian-11 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
      apt-get update
      apt-get install apache2 -y
      service apache2 restart
      echo "
<h3>Web Server: www1</h3>" | tee /var/www/html/index.html'    <= change for www2, www3
```

2. Create firewall rules to allow external traffic to reach VMs  
`gcloud compute firewall-rules create www-firewall-network-lb --target-tags network-lb-tag --allow tcp:80`
3. To verify VMs running and can be reached from outside Google Cloud  
list instances to get external IP addresses `gcloud compute instances list`  
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
4. Continuously send data over url to the IP addresses  
`while true; do curl -m1 $IPADDRESS; done`  
`CTRL + c` to stop running command.  

<br>
<hr>

### Task 5. Create an HTTP load balancer
HTTP(S) load balancers are implemented on Google Front End (GFE).
  - GFEs are globally distributed, linked to Google's network infrastructure and control plane.
  - Backend VMs must be in a (managed) instance group.
    - [Managed instance group](https://cloud.google.com/compute/docs/instance-groups) runs same app on multiple identical VMs
    - workloads benefit from 
      1. recreation of VMs if crash/deleted accidentally, 
      2. application-based health check and auto-healing
      3. auto-scaling
      4. load balancing
      5. automated updates
      5. if regional (multi-zone), can protect against zonal failures
      6. support for stateful data or configuration

**Steps**
1. Create template of instances  
2. Create managed instance group  
    - use flag to indicate instance template and size of group
3. Create firewall rules.   
    - Allow ingress traffic from Google Cloud health checking systems (130.211.0.0/22 and 35.191.0.0/16) to check on load balancer and backend service.
    - Use target tag to identify VMs.
4. Reserve global static external IP address  
.....
5. Create [health checks](https://cloud.google.com/load-balancing/docs/health-checks)   
6. Create backend service to load balancer    
7. Add instance group as backend to backend service   
.....
8. Create [URL map](https://cloud.google.com/load-balancing/docs/url-map-concepts)  
    - route incoming HTTP requests to correct backend services and buckets
    - it splits domain name into hostname and path portions
    - works by matching host rules and path matcher/path rules 
9. Create target HTTP proxy that will apply above URL map  
10. Create global [forwarding rule](https://cloud.google.com/load-balancing/docs/forwarding-rule-concepts) to route incoming requests from external IP to target HTTP proxy.

```
gcloud compute instance-templates create lb-backend-template \
   --region= \
   --network=default \
   --subnet=default \
   --tags=allow-health-check \
   --machine-type=e2-medium \
   --image-family=debian-11 \
   --image-project=debian-cloud \
   --metadata=startup-script='#!/bin/bash
     apt-get update
     apt-get install apache2 -y
     a2ensite default-ssl
     a2enmod ssl
     vm_hostname="$(curl -H "Metadata-Flavor:Google" \
     http://169.254.169.254/computeMetadata/v1/instance/name)"
     echo "Page served from: $vm_hostname" | \
     tee /var/www/html/index.html
     systemctl restart apache2'
```
`gcloud compute instance-groups managed create lb-backend-group --template=lb-backend-template --size=2 --zone= `  
```
gcloud compute firewall-rules create fw-allow-health-check \
  --network=default \
  --action=allow \
  --direction=ingress \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 \
  --target-tags=allow-health-check \
  --rules=tcp:80
```
```
gcloud compute addresses create lb-ipv4-1 \
  --ip-version=IPV4 \
  --global
```
Verifying IPv4 address reserved,
```
gcloud compute addresses describe lb-ipv4-1 \
  --format="get(address)" \
  --global
```
`gcloud compute health-checks create http http-basic-check --port 80`
```
gcloud compute backend-services create web-backend-service \
  --protocol=HTTP \
  --port-name=http \
  --health-checks=http-basic-check \
  --global
```
```
gcloud compute backend-services add-backend web-backend-service \
  --instance-group=lb-backend-group \
  --instance-group-zone= \
  --global
```
`gcloud compute url-maps create web-map-http --default-service web-backend-service`
`gcloud compute target-http-proxies create http-lb-proxy --url-map web-map-http`
```
gcloud compute forwarding-rules create http-content-rule \
    --address=lb-ipv4-1\
    --global \
    --target-http-proxy=http-lb-proxy \
    --ports=80
```

### Task 6. Testing traffic sent to your instances
- In Cloud Console, `Navigation menu` > `Network services` > `Load balancing`.
- Click on load balancer(web-map-http)
- In `Backend` section, click on name of backend and confirm that VMs are `Healthy`.
- Test load balancer. http://IP_ADDRESS using load-balancer IP.
- Should see page rendered with name of instance and zone, e.g. `Page served from: lb-backend-group-xxxx`.
