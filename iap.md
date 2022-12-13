# Setup Local IDE (App) Access to Connect to Private Cloud SQL

## Objectives
Developers would like to connect to Cloud SQL which is assigned only a private IP address. For this we will use a TCP tunnel [1] and Google Cloud Identity Aware proxy, IAP [2]. The information contained may be useful for other local applications other than IDE usage. 
* Create a Cloud SQL database instance
* Create & setup a GCE instance to host the SQL Auth proxy client
* Enable Identity Aware Proxy, IAP
* Connect the local IDE


> IMPORTANT: Test this runbook in a ‘sandbox’ or other non-production project. Add, remove, and modify the steps and scripts as needed to protect your environments.


References
1. https://cloud.google.com/sdk/gcloud/reference/sql/instances/create
2. https://cloud.google.com/iap/docs

## Set environment variables
Set environment variables to ease the use of the later scripts. 
```bash
# Set gcloud configuration
gcloud config set project [project-id]
gcloud config set compute/region [region]
gcloud config set compute/zone [zone]

export PROJECT_ID=$(gcloud config get-value project)
export REGION=$(gcloud config get-value compute/region)
export ZONE=$(gcloud config get-value compute/zone)

# Resource names/values: Choose DB version [MYSQL_8_0,POSTGRES_12, etc]
export CLOUD_SQL=cloud-sql-01
export DB_VERSION=MYSQL_8_0    
export SQL_PROXY_HOST=sql-proxy-host
export NETWORK=default
```


## Create a Cloud SQL instance
Use an existing Cloud SQL instance, or create a new instance with only a private IP [3]. However, before we create the instance a connection must be made between the VPC and the Google network of Cloud SQL.

```bash
# Enable the service if not already
gcloud services enable sqladmin.googleapis.com
gcloud services enable compute.googleapis.com
gcloud services enable servicenetworking.googleapis.com


# Ensure private Google service access is enabled on the subnet
export SUBNET=https://www.googleapis.com/compute/v1/projects/$PROJECT_ID/regions/$REGION/subnetworks/default

gcloud compute networks subnets update $SUBNET \
--enable-private-ip-google-access


# Config addresses to the Cloud SQL VPC peering, /24 is smallest
gcloud compute addresses create google-managed-services-$NETWORK \
--global \
--purpose=VPC_PEERING \
--addresses=192.168.0.0 \
--prefix-length=24 \
--network=$NETWORK


# Create the VPC peering using the address range above
gcloud services vpc-peerings connect \
--service=servicenetworking.googleapis.com \
--ranges=google-managed-services-$NETWORK \
--network=$NETWORK \
--project=$PROJECT_ID
# Output should indicate operation successful


# Create the Cloud SQL instance
gcloud beta sql instances create $CLOUD_SQL \
--database-version=$DB_VERSION \
--cpu=2 \
--memory=7680MB \
--region=$REGION \
--network=$NETWORK \
--no-assign-ip
```



References
https://cloud.google.com/sdk/gcloud/reference/sql/instances/create
https://cloud.google.com/vpc/docs/configure-private-services-access#procedure

## Setup the SQL Auth proxy
Create a GCE instance [5] to host the SQL Auth proxy application. Only a private IP is needed as IAP will be used to connect. However, this example starts with a public address and later removes it to ease the installation of the SQL Auth app [6].
Objectives
1. Create VM with only private IP
2. Ensure VM has Cloud SQL enabled in scopes
3. Install SQL Proxy
4. Start SQL proxy to listen on 0.0.0.0:3306 (or correct port for database)

Create a GCE virtual machine. Ensure that Cloud SQL is enabled in the API access scopes. Remove the default public IP address if one is used.

```bash
# Create a VM to host the SQL Proxy app
export PORT=3306 # 3306, 5432 or other port


gcloud compute instances create $SQL_PROXY_HOST \
--image=debian-10-buster-v20210420 --image-project=debian-cloud \
--zone=$ZONE \
--machine-type=e2-micro \
--subnet=default \
--scopes=https://www.googleapis.com/auth/sqlservice.admin,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/trace.append,https://www.googleapis.com/auth/devstorage.read_only \
--metadata startup-script='#! /bin/bash
sudo apt-get install wget --yes
wget https://dl.google.com/cloudsql/cloud_sql_proxy.linux.amd64 -O cloud_sql_proxy
chmod +x cloud_sql_proxy
cat <<EOF2 > cloud_sql_proxy_start.sh 
/./cloud_sql_proxy -instances='"$PROJECT_ID"':'"$REGION"':'"$CLOUD_SQL"'=tcp:0.0.0.0:'"$PORT"'
EOF2
chmod +x cloud_sql_proxy_start.sh
nohup /./cloud_sql_proxy_start.sh & 
EOF'


# Remove the ephemeral public IP
export EIP=$(gcloud compute instances describe $SQL_PROXY_HOST --format="value(networkInterfaces.accessConfigs[0].name)")

gcloud compute instances delete-access-config $SQL_PROXY_HOST --access-config-name $EIP
```

Optionally, test Cloud SQL access on the SQL proxy host machine. Open an ssh shell to the VM and use one of the commands below. 
### PostgreSQL
```bash
# PostgreSQL: Install database client as needed

psql -h $CLOUD_SQL -p [PORT] -d [DATABASE] -u [USERNAME]
\l
```
### MySQL
```bash 
# MySQL: Install database client as needed
# - https://dev.mysql.com/doc/refman/8.0/en/connecting.html

mysql -u USERNAME -p PASSWORD -h [CLOUD_SQL_IP] --port [PORT]
mysql show databases;
```

> Note: As an alternative to installing SQL Auth directly on a GCE instance OS, the SQL Auth Docker container may be used.

> Note: Optionally, use systemd to ensure the cloud_sql_proxy app is always running. Examples are
https://github.com/GoogleCloudPlatform/cloudsql-proxy/issues/4
https://www.jhanley.com/google-cloud-sql-proxy-installing-as-a-service-on-gce/


References
https://cloud.google.com/sdk/gcloud/reference/compute/instances/create
https://github.com/GoogleCloudPlatform/cloudsql-proxy

## Setup Identity Aware Proxy, IAP
If not already enabled, enable the Cloud Identity Aware Proxy API in the project.
```bash
# Enable IAP

gcloud services enable iap.googleapis.com --project $PROJECT_ID


# Create a firewall rule for IAP 
gcloud compute firewall-rules create $NETWORK-allow-from-iap \
  --direction=INGRESS \
  --action=allow \
  --rules=tcp:$PORT \
  --source-ranges=35.235.240.0/20


# Grant permissions to use IAP
export EMAIL=[email]
gcloud projects add-iam-policy-binding $PROJECT_ID \
--member=user:$EMAIL \
--role=roles/iap.tunnelResourceAccessor
```

>Troubleshooting Note: Ensure that the users (developers) have the ‘IAP-secured Tunnel User’ role assigned in IAP. Better, use a group instead of individual identities.



## Start the local app (IDE)
Objectives
1. Start a local IAP tunnel
2. Test the connection with the local application (IDE) 

Start the TCP tunnel on the local (developer) machine. Use a database client or application to connect on localhost:$PORT. 

```bash
# Start tunnel on local machine

export PORT=3306 # 3306, 5432 or other port

gcloud compute start-iap-tunnel $SQL_PROXY_HOST $PORT \
--zone $ZONE \
--local-host-port=localhost:$PORT
```

> Troubleshooting Note: If unable to connect
Ensure the firewall is configured
Ensure the cloud_sql_proxy is running successfully on the GCE host


References
https://cloud.google.com/sdk/gcloud/reference/compute/start-iap-tunnel
