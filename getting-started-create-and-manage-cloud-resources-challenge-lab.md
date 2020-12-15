# Getting Started: Create and Manage Cloud Resources: Challenge Lab

## GSP313

## Overview

You must complete a series of tasks within the allocated time period. Instead of following step-by-step instructions, you'll be given a scenario and a set of tasks: you figure out how to complete it on your own! An automated scoring system (shown on this page) will provide feedback on whether you have completed your tasks correctly.

To score 100%, you must complete all tasks within the time period!

When you take a Challenge Lab, you will not be taught Google Cloud concepts. To build the solution to the challenge presented, use skills learned from the labs in the Quest this challenge lab is part of. You are expected to extend your learned skills and to change default values, but new concepts are not introduced.

> This lab is recommended only for students who have completed the labs in the Getting Started: Create and Manage Cloud Resources Quest. Be sure to review those labs before starting this lab. Are you ready for the challenge?

Topics tested:

- Create an instance

- Create a 3-node Kubernetes cluster and run a simple service

- Create an HTTP(s) load balancer in front of two web servers

### Setup

#### Before you click the Start Lab button

Read these instructions. Labs are timed and you cannot pause them. The timer, which starts when you click Start Lab, shows how long Google Cloud resources will be made available to you.

This Qwiklabs hands-on lab lets you do the lab activities yourself in a real cloud environment, not in a simulation or demo environment. It does so by giving you new, temporary credentials that you use to sign in and access Google Cloud for the duration of the lab.

# What you need

To complete this lab, you need:

- Access to a standard internet browser (Chrome browser recommended).
- Time to complete the lab.

> Note: If you already have your own personal Google Cloud account or project, do not use it for this lab.

> Note: If you are using a Pixelbook, open an Incognito window to run this lab.

### Activate Cloud Shell

Cloud Shell is a virtual machine that is loaded with development tools. It offers a persistent 5GB home directory and runs on the Google Cloud. Cloud Shell provides command-line access to your Google Cloud resources.

- In the Cloud Console, in the top right toolbar, click the **Activate Cloud Shell** button.


- Click Continue.

It takes a few moments to provision and connect to the environment. When you are connected, you are already authenticated, and the project is set to your PROJECT_ID. For example:

`gcloud` is the command-line tool for Google Cloud. It comes pre-installed on Cloud Shell and supports tab-completion.

You can list the active account name with this command:

```bash
    gcloud auth list
```

(Output)

```bash
Credentialed accounts:
 - <myaccount>@<mydomain>.com (active)
```

(Example output)

```bash
Credentialed accounts:
 - google1623327_student@qwiklabs.net
```

You can list the project ID with this command:

```bash
gcloud config list project
```

(Output)

```bash
[core]
project = <project_ID>
```

(Example output)

```bash
[core]
project = qwiklabs-gcp-44776a13dea667a6
```

> For full documentation of `gcloud` see the [gcloud command-line tool overview](https://cloud.google.com/sdk/gcloud).


## Challenge scenario

You have started a new role as a Junior Cloud Engineer for Jooli, Inc. You are expected to help manage the infrastructure at Jooli. Common tasks include provisioning resources for projects.

You are expected to have the skills and knowledge for these tasks, so step-by-step guides are not provided.

Some Jooli, Inc. standards you should follow:

- Create all resources in the default region or zone, unless otherwise directed.

- Naming normally uses the format team-resource; for example, an instance could be named **nucleus-webserver1**.

- Allocate cost-effective resource sizes. Projects are monitored, and excessive resource use will result in the containing project's termination (and possibly yours), so plan carefully. This is the guidance the monitoring team is willing to share: unless directed, use **f1-micro** for small Linux VMs, and use **n1-standard-1** for Windows or other applications, such as Kubernetes nodes.

### Your challenge

As soon as you sit down at your desk and open your new laptop, you receive several requests from the Nucleus team. Read through each description, and then create the resources.

### Task 1: Create a project jumphost instance

You will use this instance to perform maintenance for the project.

Requirements:

- Name the instance **nucleus-jumphost**.
- Use an **f1-micro** machine type.
- Use the default image type (Debian Linux).

### Solution 1

```bash
# Create a project jumphost instance

gcloud compute instances create nucleus-jumphost \
	--machine-type f1-micro \
	--image-family debian-9 \
	--zone us-east1-b \
	--image-project debian-cloud 
```

### Task 2: Create a Kubernetes service cluster

The team is building an application that will use a service running on Kubernetes. You need to:

- Create a cluster (in the **us-east1-b** zone) to host the service.
- Use the Docker container hello-app (`gcr.io/google-samples/hello-app:2.0`) as a place holder; the team will replace the container with their own work later.
- Expose the app on port 8080.

### Solution 2

```bash
# Create a Kubernetes service cluster
gcloud container clusters create nucleus-cluster \
          --num-nodes 1 \
          --region us-east1

# configure cloud shell with GKE credentials
gcloud container clusters get-credentials nucleus-cluster \
	--region us-east1

# Deploy sample image
kubectl create deployment hello-server \
	--image=gcr.io/google-samples/hello-app:2.0

# Expose image on load balancer on dedicated port
kubectl expose deployment hello-server \
          --type=LoadBalancer \
          --port 8080
```

### Task 3: Set up an HTTP load balancer

You will serve the site via nginx web servers, but you want to ensure that the environment is fault-tolerant. Create an HTTP load balancer with a managed instance group of 2 nginx web servers. Use the following code to configure the web servers; the team will replace this with their own configuration later.

```bash
cat << EOF > startup.sh
#! /bin/bash
apt-get update
apt-get install -y nginx
service nginx start
sed -i -- 's/nginx/Google Cloud Platform - '"\$HOSTNAME"'/' /var/www/html/index.nginx-debian.html
EOF
```

You need to:

- Create an instance template.
- Create a target pool.
- Create a managed instance group.
- Create a firewall rule to allow traffic (80/tcp).
- Create a health check.
- Create a backend service, and attach the managed instance group.
- Create a URL map, and target the HTTP proxy to route requests to your URL map.
- Create a forwarding rule.

### Solution 3

```bash
# Create  the startup script
cat << EOF > startup.sh
#! /bin/bash
apt-get update
apt-get install -y nginx
service nginx start
sed -i -- 's/nginx/Google Cloud Platform - '"\$HOSTNAME"'/' /var/www/html/index.nginx-debian.html
EOF

# Create instance template
gcloud compute instance-templates create web-server-template \
          --metadata-from-file startup-script=startup.sh \
          --machine-type g1-small \
          --region us-east1

# Create instance pool
gcloud compute instance-groups managed create web-server-group \
          --base-instance-name web-server \
          --size 2 \
          --template web-server-template \
          --region us-east1

# Allow http trafic
gcloud compute firewall-rules create www-firewall --allow tcp:80

# Create healthcheck
gcloud compute http-health-checks create http-basic-check


gcloud compute instance-groups managed \
          set-named-ports web-server-group \
          --named-ports http:80 \
          --region us-east1
		  
gcloud compute backend-services create web-server-backend \
          --protocol HTTP \
          --http-health-checks http-basic-check \
          --global

# Create a backend service, and attach the managed instance group.
gcloud compute backend-services add-backend web-server-backend \
          --instance-group web-server-group \
          --instance-group-region us-east1 \
          --global

# Create a URL map, and target the HTTP proxy to route requests to your URL map.
gcloud compute url-maps create web-server-map \
    --default-service web-server-backend


gcloud compute target-http-proxies create http-lb-proxy \
    --url-map web-server-map


# Create a forwarding rule.
gcloud compute forwarding-rules create http-content-rule \
        --global \
        --target-http-proxy http-lb-proxy \
        --ports 80


gcloud compute forwarding-rules list

```