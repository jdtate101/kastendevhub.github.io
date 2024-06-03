---
layout: post
title: The secret of short RTO is automated DR 
description: When disaster strikes no need to say that panic also and any restore operation becomes stressful and error prone. Being able to automate entirely your restore operation is the key. Not only it speeds up your action and allow parrallelisation it also gives you a real measure of your RTO.
date: 2024-05-23 13:48:00 +0000
author: michaelcourcy
image: '/images/posts/2024-05-23-automated-dr-with-kasten/automated-dr.jpg'
image_caption: 'Automate your DR completly'
tags: [Disaster recovery, Automation, Kasten, RTO]
featured:
---
# What is disater recovery at Kasten 

At kasten we have a very opiniated view of what should be disaster recovery. We are very different from the classical
backup tool because we protect application not infrastructure.

Protecting infrastructure (for instance doing a backup of VM) in a cloud native world does not make sense any more when infrastructure as code is becoming the "de facto". But you still need to protect your buisness data and your buisness metadata.

So disaster recovery comes in 2 flavors at Kasten: 
- You rebuild a new cluster, you reinstall Kasten and you restore the apps : known as the rebuild strategy.
- You continuously restore the apps in a standby cluster : known as the replicate strategy.

In this post we are going to focus on the first one : the rebuild strategy.

# Automate the complete restoration of your cluster

In this example we'll demonstrate an automatic revovery from a disaster. Disaster will emulate this scenario : 
- All applications removed
- Kasten removed, kasten-io namespace deleted 

And the  script `dr.sh` will :
- Reinstall Kasten 
- Restore Kasten Catalog, all profiles and all policies
- Restore all the namespaces in parrallel

This is only possible because Kasten is fully API oriented and designed with automation as a first preoccupation. 
Being able to completly automate the restoration of all the namespaces in any new cluster has a strong impact on your RTO. 

# Prerequisite 

## Create a cluster with kasten installed

We need a kubernetes cluster with Kasten installed and a location profile.

Follow for instance this [tutorial](./kasten-on-eks-1).


## Create multiple applications  

### Pacman 

If you followed the [previous tutorial](./kasten-on-eks-1) Pacman
is already installed, otherwise follow these steps.

Create the application and play one or two games to create some data in the mongodb. 
```
helm repo add pacman https://shuguet.github.io/pacman/
helm install pacman pacman/pacman -n pacman --create-namespace
kubectl port-forward -n pacman svc/pacman 8081:80
# open http://localhost:8081
```

Create an hourly policy (backup + export) for pacman and run it once

### Mysql 

Create the database and some data 
```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install mysql-release bitnami/mysql --namespace mysql --create-namespace \
    --set auth.rootPassword=rooroot
kubectl exec -n mysql sts/mysql-release -it -- bash
mysql --user=root --password=rooroot
CREATE DATABASE test;
use test;
CREATE TABLE pets (name VARCHAR(20), owner VARCHAR(20), species VARCHAR(20), sex CHAR(1), birth DATE, death DATE);
INSERT INTO pets VALUES ('Puffball','Diane','hamster','f','1999-03-30',NULL);
SELECT * FROM pets;
exit
exit
```

Create a hourly policy (backup + export) for mysql and launch it once.


### postgres 

Create the database and add some data 
```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install my-release --namespace postgres --create-namespace bitnami/postgresql
export POSTGRES_PASSWORD=$(kubectl get secret --namespace postgres my-release-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)
kubectl run my-release-postgresql-client --rm --tty -i --restart='Never' --namespace postgres \
  --image docker.io/bitnami/postgresql:16.2.0-debian-12-r18 \
  --env="PGPASSWORD=$POSTGRES_PASSWORD" \
  --command -- psql --host my-release-postgresql -U postgres -d postgres -p 5432
CREATE DATABASE test;
\c test
CREATE TABLE COMPANY(
     ID INT PRIMARY KEY     NOT NULL,
     NAME           TEXT    NOT NULL,
     AGE            INT     NOT NULL,
     ADDRESS        CHAR(50),
     SALARY         REAL,
     CREATED_AT    TIMESTAMP
);
INSERT INTO COMPANY (ID,NAME,AGE,ADDRESS,SALARY,CREATED_AT) VALUES (10, 'Paul', 32, 'California', 20000.00, now());
select * from company;
\q
```

Create an hourly policy (backup + export) for postgres and run it once

### Elasticsearch 

```
helm repo add elastic https://helm.elastic.co
helm install --namespace elastic elasticsearch elastic/elasticsearch \
  --set antiAffinity=soft --create-namespace
kubectl exec -it elasticsearch-master-0 -n elastic -c elasticsearch -- bash
curl -X PUT "https://elastic:${ELASTIC_PASSWORD}@localhost:9200/customer?pretty" -k
curl -X PUT "https://elastic:${ELASTIC_PASSWORD}@localhost:9200/customer/_doc/1?pretty" \
  -H 'Content-Type: application/json' -d '{"name": "John Smith"}' -k
curl -X GET "https://elastic:${ELASTIC_PASSWORD}@localhost:9200/_cat/indices?v" -k
curl -X GET "https://elastic:${ELASTIC_PASSWORD}@localhost:9200/customer/_search?q=*&pretty" -k
exit
```

Create an hourly policy (backup + export) for postgres and run it once

# Prepare for disastser recovery 

Activate disaster recovery in `Settings > Disaster recovery`, and run once the policy `k10-disaster-recovery-policy`.

We run it once because we need at least one backup of the Kasten catalog, but by default this policy run hourly. 

## Save the kasten values in a file

```
helm get values k10 -n kasten-io > k10-values.yaml
```

Add the eula it's necessary to run the disaster recovery as for any operation eula is mandatory.
```
cat<<EOF >> k10-values.yaml
eula:
  accept: true
  company: kasten
  email: michael@kasten.io
EOF
```

## Create a dr.yaml file to store necessary informations for restoring the catalog 

This file should be kept securely as it allow to restore all the applications in any cluster. 
It will contain the following information: 
- profile used for the disaster recovery policy 
- Credential to access this profile 
- The passphrase used when creating the disaster recovery policy 
- the uid of the cluster 

```
profile=$(kubectl get policies.config.kio.kasten.io k10-disaster-recovery-policy -o jsonpath='{.spec.actions[0].backupParameters.profile.name}')
profile_secret=$(kubectl get -n kasten-io profiles.config.kio.kasten.io $profile -o jsonpath='{.spec.locationSpec.credential.secret.name}')
kubectl get -n kasten-io secret $profile_secret -o yaml > dr.yaml
echo "---" >> dr.yaml
kubectl get -n kasten-io profiles.config.kio.kasten.io $profile -o yaml >> dr.yaml
echo "---" >> dr.yaml
kubectl get secret -n kasten-io k10-dr-secret -o yaml >> dr.yaml
echo "---" >> dr.yaml
uid=$(kubectl get namespace default -o jsonpath="{.metadata.uid}")
kubectl create configmap -n kasten-io previous-cluster --from-literal=uidcluster=$uid --dry-run=client -o yaml >> dr.yaml
``` 

## Simulate a disaster 

delete all the application namespace and remove kasten 
```
kubectl delete ns postgres \
                  pacman \
                  elastic \
                  mysql
helm uninstall k10 -n kasten-io 
kubectl delete ns kasten-io 
```

# Execute the complete restore

```
wget https://raw.githubusercontent.com/michaelcourcy/k10-automatic-dr/main/dr.sh
chmod +x dr.sh 
./dr.sh
```

This script will 
- Do a fresh install of Kasten
- Recreate the profile and the DR pass phrase
- Restore the catalog of the previous cluster 
- Restore in parrallel the 4 namespaces : postgres, pacman, elastic and mysql 

# check all data are back

For pacman check the high scores.

For mysql 
```
kubectl exec -n mysql sts/mysql-release -it -- bash
mysql --user=root --password=rooroot
use test;
SELECT * FROM pets;
exit
exit
```

For postgres
```
export POSTGRES_PASSWORD=$(kubectl get secret --namespace postgres my-release-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)
kubectl run my-release-postgresql-client --rm --tty -i --restart='Never' --namespace postgres \
  --image docker.io/bitnami/postgresql:16.2.0-debian-12-r18 \
  --env="PGPASSWORD=$POSTGRES_PASSWORD" \
  --command -- psql --host my-release-postgresql -U postgres -d postgres -p 5432
\c test
select * from company;
\q
```

For elastic search 
```
kubectl exec -it elasticsearch-master-0 -n elastic -c elasticsearch -- bash
curl -X GET "https://elastic:${ELASTIC_PASSWORD}@localhost:9200/_cat/indices?v" -k
curl -X GET "https://elastic:${ELASTIC_PASSWORD}@localhost:9200/customer/_search?q=*&pretty" -k
exit
```

# Conclusion 

This example show how a such complex operation of restoring kasten and all the apps in a cluster become so easy 
when you can automate everything.
Testing you disaster recovery procedure is very important and automation of thoses operations should be 
your first preoccupation. Working with a tool like Kasten that regards API as first class citizen is the key to success.