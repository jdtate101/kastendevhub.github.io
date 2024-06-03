---
author: michaelcade
date: 2024-04-17 16:21:37 +0100
description: "Extending Kasten with the Veeam Data Platform APIs"
featured: false
image: "/images/posts/2024-04-17-Extending-k10-vbAWS/1-vbaws-pre-12-1.png"
image_caption: "Extending Kasten with the Veeam Data Platform APIs"
layout: post
published: true
tags: [veeam, api, amazon, rds]
title: "Extending K10 to Drive Veeam Backup for AWS"
---

# Extending K10 to Drive Veeam Backup for AWS 

This is still a little work in progress project but wanted to share as it really highlights some of the important features of both the Veeam Data Platform and the extensibility of Kasten.

## The Use Case 

For those familiar with Kanister you will know that this is a way for Kasten to interact with data services inside and outside of the Kubernetes cluster, we have examples that allow Kasten to protect Amazon RDS and I have seen other use cases with AzureSQL.  

For those not so familiar, you can find out more here - https://www.kanister.io/ 

Kanister is an open-source project that has been donated to the CNCF Landscape as a sandbox project. Kasten uses Kanister for the above use cases alongside the ability to protect the associated Kubernetes objects of your complete application. 

The specific use case here was to create something that would interact initially with one area of the Veeam Data Platform, I chose Veeam Backup for AWS as a starting point and when this project started, when Veeam Data Platform was pre 12.1. I wanted to cover the art of the possible and what would be available with 12.1 in theory. 

I wanted to have the ability to use a Kanister blueprints via Kasten to orchestrate a complete backup of an RDS instance associated to my application in my Kubernetes cluster but really this could also include EC2, VPC, EFS as well any new services that come available within the Veeam Backup for AWS API down the line. This would also pave the way for further Veeam API art of the possible ideas regarding interacting for example with Veeam Backup and Replication and Veeam Backup for Microsoft Azure. 

## The Beginning 
As I mentioned I wanted to start with the pre 12.1 of the Veeam Data Platform. 

In the diagram below we can see that we have a Kubernetes cluster, this is where Kasten is deployed. We will also have an application namespace, secret and the Kanister blueprint. 

We then also have Veeam Backup for AWS deployed and for our example we have prebuilt a backup policy that protects our workloads (RDS Instance). 

![Logical Diagram of Kasten Connected to Veeam Backup for AWS](/images/posts/2024-04-17-Extending-k10-vbAWS/1-vbaws-pre-12-1.png)

For this example, we do not have anything running inside of the application namespace, but we can imagine there being a web front end application that connects to the AWS RDS instance. The above enables us to have part of our application within the Kubernetes cluster and another part external leveraging AWS services. We want to protect everything associated with the application as one. 

## What do we need? 
At a high level, we need the following to get this up and running: 

-	Veeam Backup for AWS deployed in our AWS environment. 
-	A policy created within our deployed Veeam Backup for AWS. 
-	An RDS Instance. 
-	Kasten deployed within a Kubernetes cluster. (This does not need to be in AWS)
-	An application namespace. 
-	A Secret to communicate with VBAWS. 
-	Our blueprint deployed and secret annotated. 

### Deploy Veeam Backup for AWS (VBAWS)

Veeam Backup for AWS (VBA) comes as a pre-packaged appliance available via the AWS marketplace or as a standalone AMI.

{: .alert-info }
Note that with 12.1 the deployment options for new features now requires Veeam Backup & Replication but these API endpoints remain the same

To learn more, it's worth reading the latest version of the [Veeam Backup for AWS user guide](https://helpcenter.veeam.com/docs/vbaws/guide/overview.html).

We do also have Terraform code available for deployment on [VeeamHUB](https://github.com/VeeamHub/veeam-terraform/tree/master/veeam-backup-aws)

### Create a Policy & RDS Instance 

Our blueprint is going to start an existing policy, we are not going to be creating a policy through Kasten.

For this example we are looking to protect an RDS instance (Again, we will need also deploy an RDS instance within our AWS Account as mentioned above)

The policy created within Veeam Backup for AWS enables us to protect our RDS instance.

Veeam Backup for AWS creates a cloud-native snapshot for each RDS resource added to a backup policy. The cloud-native snapshot is further used to create a snapshot replica in another AWS Region or another AWS account.

You can follow this [guide](https://helpcenter.veeam.com/docs/vbaws/guide/policies_create_rds.html?ver=6a) on how to create an RDS policy. 

At this stage we should have: 

- A deployment of Veeam Backup for AWS 
- An RDS Instance 
- An RDS policy with an on-demand schedule 

We will also need to ensure that via the security group associated to our Veeam Backup for AWS appliance, we have access to
the Veeam Backup for AWS appliance from our Kubernetes cluster on Port 11005, which is the default API port, although worth noting that this can be changed. 

### Deploy Kasten 

Now that we have our AWS environment configured, let's navigate to our Kubernetes cluster, to deploy and configure Kasten. We won't cover how to install Kasten in a K8s cluster, but you can reference the [Kasten install documentation](https://docs.kasten.io/latest/install/index.html) for that.

### The Blueprint 

The blueprint that provides the instructions can be found below, currently this blueprint only contains the backup phase and not the recovery steps. We can add this to our Kubernetes cluster using the following: 

`kubectl create -f vbaws-bp.yaml`

```
apiVersion: cr.kanister.io/v1alpha1
kind: Blueprint
metadata:
  name: vbaws-bp
  namespace: kasten-io
actions:
  backup:
    phases:
    - func: KubeTask
      name: trigger-vbaws-policy      
      args:
        namespace: "{{ .Object.metadata.namespace }}"
        image: "badouralix/curl-jq"
        command:
        - sh
        - -o
        - errexit
        - -o
        - pipefail
        - -c
        - |
          # First grab the token 
          # create an access_token
          url="{{ index .Object.data "vbaws-url" | toString | b64dec }}"
          username="{{ index .Object.data "username" | toString | b64dec }}"
          password="{{ index .Object.data "password" | toString | b64dec }}"
          policy="{{ index .Object.data "policy" | toString | b64dec }}"
          access_token=$(curl -k -X 'POST' \
            "https://$url/api/v1/token" \
            -H 'accept: application/json' \
            -H 'x-api-version: 1.4-rev0' \
            -H 'Content-Type: application/x-www-form-urlencoded' \
            -d "grant_type=password&username=$username&password=$password" | jq -r '.access_token')

          # launch the policy with the access token and grab the session_id 
          session_id=$(curl -k -X 'POST' \
            "https://$url/api/v1/$policy/start" \
              -H 'accept: application/json' \
              -H 'x-api-version: 1.4-rev0' \
              -H "Authorization: Bearer $access_token"  -d '' | jq -r '.sessionId')

          # follow the execution with the session_id 
          status=$(curl -k -X 'GET' \
          "https://$url/api/v1/sessions/$session_id" \
          -H 'accept: application/json' \
          -H 'x-api-version: 1.4-rev0' \
          -H "Authorization: Bearer $access_token"  | jq -r '.status')

          while test "Succeeded" != "$status"
          do 
            echo "Status of the session $session_id is not Suceeded $status"
            status=$(curl -k -X 'GET' \
              "https://$url/api/v1/sessions/$session_id" \
              -H 'accept: application/json' \
              -H 'x-api-version: 1.4-rev0' \
              -H "Authorization: Bearer $access_token"  | jq -r '.status')
            sleep 3   
          done 

          exit 0
```

### Create a Secret

Now that we have Veeam Backup for AWS and Kasten deployed and configured, we can continue to create the objects to communicate between the two platforms. 

We need to create a secret within our Kubernetes cluster to store our credentials, which are referenced in our blueprint above. For simplicity, we should first create a new namespace. 

`kubectl create ns kasten-vbaws`

Our secret should then contain the following: 

```
kubectl create secret generic vbaws-policy -n kasten-vbaws \
    --from-literal=vbaws-url=<ip>:<port> \
    --from-literal=username=<username> \
    --from-literal=password=<password> \
    --from-literal=policy=<policy>
```

We will first need the Veeam Backup for AWS IP address and port. You can find this information in your AWS management console.
The default port is `11005` (although worth noting that this can be customized, so ensure that the port matches what was configured in Veeam Backup for AWS).

![](/images/posts/2024-04-17-Extending-k10-vbAWS/2-vbaws-public-ip.jpg)

When deploying our Veeam Backup for AWS appliance, we need to define a `username` and `password`, which need to be specified in the secret above.

Finally we must get the ID of the policy that we have created and that we wish to trigger from Kasten.

We can obtain this through our Veeam Backup for AWS console by selecting the export to CSV or XML action and the ID will be visible,
and must be specified in the secret. 

![Veeam Backup for AWS Policy ID](/images/posts/2024-04-17-Extending-k10-vbAWS/3-vbaws-policy.jpg)

We should then add the API call to our policy, for RDS it will look like this - `rds/policies/16f23f8b-a811-4d62-976f-91c36c415f52`

Also tested against `virtualMachines/policies/` for EC2 snapshot backups. 

![Triggering a Veeam Backup for AWSpolicy via Kasten](/images/posts/2024-04-17-Extending-k10-vbAWS/4-vbaws-kasten-backup.jpg)

Not yet tested but we should also be able to use: 

`efs/policies/`
`vpc/policy/`

We can then create a secret in our Kubernetes cluster and ensure that we create it in the correct namespace (i.e. kasten-vbaws)

In order for Kasten to use our blueprint and secret we have to annotate a resource/object in our application namespace.

In this example, we will annotate our secret. 

`kubectl annotate secret vbaws-policy kanister.kasten.io/blueprint='vbaws-bp' -n kasten-vbaws`

## Where else could this go? 

There were some significant changes in 12.1 of the Veeam Data Platform which includes more services available to be protected by Veeam Backup for AWS such as DynamoDB and Object Storage as a primary data source. 

I have further ideas on showing more extensibility like this when it comes to Virtual Machine backup especially when you now see Kubernetes being a first class citizen when it comes to running VMs on top of Kubernetes. I also have some other external data services I will cover in further posts. 

![](/images/posts/2024-04-17-Extending-k10-vbAWS/5-vbaws-post-12-1.png)
