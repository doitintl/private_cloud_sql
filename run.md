# Accessing database from Cloud Run

Since Cloud Run doesn't run within your VPC, as of today, then it cannot see a database with only a private IP address.  Fortunately, Google has provided a VPC Connector that can help Functions and Cloud Run to connect to resources within your VPC.  Here we will set up the required services needed to successfuly launch a Cloud Run funtion that can access the database with only a private IP address.

## Setting up Artifact Registry
> NOTE: For this, we are assuming that the application is containerized and the reader is familiar with docker.  We do have a basic command to create a container but that's as far as we go.

The first step is to set up Artifact Registry.  This is the Google managed container registry which we can use to deploy the container to Cloud Run.

```bash
# Enable Artifact Registry
gcloud services enable artifactregistry.googleapis.com

# Create a repository
export REPOSITORY=my-repository
gcloud artifacts repositories create \
$REPOSITORY --location $REGION --repository-format docker

# Enable docker to use the registry
 gcloud auth configure-docker \
    us-central1-docker.pkg.dev  

# Push the container to the new repository
export IMAGE=my-image
export TAG=dev
docker build --tag=us-central1-docker.pkg.dev/$PROJECT_ID/$REPOSITORY/$IMAGE:$TAG --file=Dockerfile .
docker push us-central1-docker.pkg.dev/$PROJECT_ID/$REPOSITORY/$IMAGE:$TAG
```
## Setting up VPC Connector
Before we create our Cloud Run function, we'll need to create the VPC Connector that will allow Cloud Run to access our VPC.

```bash
export CONNECTOR=my-connector
gcloud compute networks vpc-access connectors create $CONNECTOR \
--region=$REGION \    
--network=$NETWORK \
--range=10.8.0.0/28 \
--min-instances=2 \
--max-instances=10 \
--machine-type=e2-micro
```

## Setting up Cloud Run
The last thing we need to do is to deploy the service to Cloud Run passing in the location and credentials for the database and the VPC Connector we just created.

```bash
export DBHOST=$(gcloud sql instances describe $CLOUD_SQL --format "value(ipAddresses[0].ipAddress)")
export DBNAME=[database]
export DBUSER=[user]
export DBPASS=[password]

gcloud services enable run.googleapis.com

gcloud run deploy my-service \
--image=us-central1-docker.pkg.dev/$PROJECT_ID/$REPOSITORY/$IMAGE:$TAG \
--region=$REGION \
--ingress=all \
--max-instances=1 \
--set-env-vars=DBHOST=${DBHOST},DBNAME=${DBNAME},DBUSER=${DBUSER},DBPASS=${DBPASS} \
--vpc-connector=$CONNECTOR
```