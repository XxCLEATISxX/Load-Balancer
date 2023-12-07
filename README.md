# Load-Balancer
Project part 3

Load balancers play a crucial role in cloud computing setups, helping efficiently distribute incoming network traffic across multiple servers or computing resources. Their main job is to improve the availability and reliability of applications by preventing any single server from bearing too much load, thus avoiding performance issues and potential downtime. In simpler terms, load balancing makes sure that resources are used optimally, response times are faster, and the system can scale effectively. It's an essential part of creating strong and high-performance cloud-based systems.

For part 3 of the project, I worked on setting up and configuring load balancers in the Google Cloud Platform, specifically exploring two types: the network load balancer and the HTTP load balancer. The hands-on lab guided us through creating virtual machine instances, configuring firewall rules, setting up health checks, and implementing forwarding rules. With this guide, we should be able to understand and apply both network and HTTP load balancing scenarios, gaining practical experience in managing and optimizing cloud infrastructure for better performance and reliability. 

### Step 1
```bash
# Set the default region for Compute Engine resources to us-central1
gcloud config set compute/region us-central1

# Set the default zone for Compute Engine resources to us-central1-a
gcloud config set compute/zone us-central1-a
```
### Step 2
```bash
# Create a virtual machine www1 in the default zone (us-central1-a)
gcloud compute instances create www1 \
    --zone=us-central1-a \
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

# Create a virtual machine www2 in the default zone (us-central1-a)
gcloud compute instances create www2 \
    --zone=us-central1-a \
    --tags=network-lb-tag \
    --machine-type=e2-small \
    --image-family=debian-11 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
      apt-get update
      apt-get install apache2 -y
      service apache2 restart
      echo "
<h3>Web Server: www2</h3>" | tee /var/www/html/index.html'

# Create a virtual machine www3 in the default zone (us-central1-a)
gcloud compute instances create www3 \
    --zone=us-central1-a \
    --tags=network-lb-tag \
    --machine-type=e2-small \
    --image-family=debian-11 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
      apt-get update
      apt-get install apache2 -y
      service apache2 restart
      echo "
<h3>Web Server: www3</h3>" | tee /var/www/html/index.html'

# Create a firewall rule to allow external traffic to the VM instances
gcloud compute firewall-rules create www-firewall-network-lb \
    --target-tags network-lb-tag --allow tcp:80

# List your instances to get their external IP addresses
gcloud compute instances list

# Verify that each instance is running with curl, replacing the IP address with the IP address for each VM
curl http://35.239.44.69
curl http://34.122.211.33
curl http://34.30.143.188
```

### Step 3
```bash
# Create a static external IP address for your load balancer. This command will reserve an external IP address for your load balancer. The output will provide information about the created IP address.
gcloud compute addresses create network-lb-ip-1 --region us-central1

# Add a legacy HTTP health check resource. This command creates an HTTP health check named basic-check, which will be used to monitor the health of the backend instances in the load balancer.
gcloud compute http-health-checks create basic-check

# Add a target pool and use the health check. This command creates a target pool named www-pool in the specified region and associates it with the previously created HTTP health check. The target pool is where the load balancer will direct traffic.
gcloud compute target-pools create www-pool --region us-central1 --http-health-check basic-check

# Add instances to the target pool. This command adds the instances to the target pool. The load balancer will distribute incoming traffic among these instances.
gcloud compute target-pools add-instances www-pool --instances www1,www2,www3

# Add a forwarding rule. This command creates a forwarding rule named www-rule in the specified region. It associates the rule with the external IP address, directs traffic to the target pool, and specifies that it should handle traffic on port 80.
gcloud compute forwarding-rules create www-rule --region us-central1 --ports 80 --address network-lb-ip-1 --target-pool www-pool
```

### Step 4
```bash
# View the external IP address of the forwarding rule. This command retrieves and displays detailed information about the specified forwarding rule, including its external IP address.
gcloud compute forwarding-rules describe www-rule --region us-central1

# Access the external IP address and show it. These commands capture the external IP address of the forwarding rule in a variable and then display the IP address.
IPADDRESS=$(gcloud compute forwarding-rules describe www-rule --region us-central1 --format="json" | jq -r .IPAddress)

echo $IPADDRESS

# Use curl command to access the external IP address. This command continuously sends HTTP requests to the external IP address using curl, providing a way to observe the load balancer's behavior by seeing the responses from different instances.
while true; do curl -m1 $IPADDRESS; done
```

### Step 5
```bash
# Create a load balancer template. This command creates an instance template (lb-backend-template) with configurations for the managed instance group.
gcloud compute instance-templates create lb-backend-template \
   --region=us-central1 \
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

# Create a managed instance group. This command creates a managed instance group based on the previously created template, specifying the size and zone.
gcloud compute instance-groups managed create lb-backend-group \
   --template=lb-backend-template --size=2 --zone=us-central1-a

# Create the fw-allow-health-check firewall rule. This command creates a firewall rule to allow health check traffic from specified source ranges.
gcloud compute firewall-rules create fw-allow-health-check \
  --network=default \
  --action=allow \
  --direction=ingress \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 \
  --target-tags=allow-health-check \
  --rules=tcp:80

# Create a global static external IP address. This command reserves a global static external IP address for the load balancer.
gcloud compute addresses create lb-ipv4-1 \
  --ip-version=IPV4 \
  --global

# Note the reserved IPv4 address. This command retrieves and displays the reserved IPv4 address for future reference.
gcloud compute addresses describe lb-ipv4-1 \
  --format="get(address)" \
  --global

# Create a health check for the load balancer. This command creates an HTTP health check on port 80 for checking backend instance health.
gcloud compute health-checks create http http-basic-check --port 80

# Create a backend service. This command creates a backend service for HTTP on port 80, using the health check.
gcloud compute backend-services create web-backend-service \
  --protocol=HTTP \
  --port-name=http \
  --health-checks=http-basic-check \
  --global

# Add the instance group as the backend to the backend service. This command associates the managed instance group with the backend service in the specified zone.
gcloud compute backend-services add-backend web-backend-service \
  --instance-group=lb-backend-group \
  --instance-group-zone=us-central1-a \
  --global

# Create a URL map to route incoming requests to the default backend service. This command creates a URL map to define how requests should be routed to different backend services.
gcloud compute url-maps create web-map-http --default-service web-backend-service

# Create a target HTTP proxy to route requests to the URL map. This command creates a target HTTP proxy to link the URL map to the proxy.
gcloud compute target-http-proxies create http-lb-proxy --url-map web-map-http

# Create a global forwarding rule to route incoming requests to the proxy. This command creates a global forwarding rule associating it with the global external IP address and specifying the target HTTP proxy for routing traffic. The rule listens on port 80.
gcloud compute forwarding-rules create http-content-rule \
   --address=lb-ipv4-1 \
   --global \
   --target-http-proxy=http-lb-proxy \
   --ports=80
```

### Step 6
```bash
# Open the Load Balancing page in the Google Cloud console
# Navigate to Network services > Load balancing

# Click on the load balancer (web-map-http) you created

# In the Backend section, click on the name of the backend
# Confirm that the VMs are Healthy. If not, wait a few moments and try reloading the page.

# When VMs are healthy, test the load balancer using a web browser
# Replace IP_ADDRESS with the load balancer's IP address
# This may take three to five minutes. If you do not connect, wait a minute, and then reload the browser.
# Your browser should render a page with content showing the name of the instance that served the page,
# along with its zone (for example, Page served from: lb-backend-group-xxxx).
```
Congratulations! if you followed these instructions, you should now have balance loaders to work with!




















