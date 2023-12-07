# Load-Balancer
Project part 3

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

# Verify that each instance is running with curl, replacing [IP_ADDRESS] with the IP address for each VM
curl http://35.239.44.69
curl http://34.122.211.33
curl http://34.30.143.188
```

### Step 3
```bash
# Create a static external IP address for your load balancer. This command will reserve an external IP address for your load balancer. The output will provide information about the created IP address.
gcloud compute addresses create network-lb-ip-1 --region=us-central1

# Add a legacy HTTP health check resource. This command creates an HTTP health check named basic-check, which will be used to monitor the health of the backend instances in the load balancer.
gcloud compute http-health-checks create basic-check

# Add a target pool and use the health check. This command creates a target pool named www-pool in the specified region and associates it with the previously created HTTP health check. The target pool is where the load balancer will direct traffic.
gcloud compute target-pools create www-pool --region=us-central1 --http-health-check=basic-check

# Add instances to the target pool. This command adds the instances to the target pool. The load balancer will distribute incoming traffic among these instances.
gcloud compute target-pools add-instances www-pool --instances=www1,www2,www3

# Add a forwarding rule. This command creates a forwarding rule named www-rule in the specified region. It associates the rule with the external IP address, directs traffic to the target pool, and specifies that it should handle traffic on port 80.
gcloud compute forwarding-rules create www-rule --region=us-central1 --ports=80 --address=network-lb-ip-1 --target-pool=www-pool

```








