## Setting up Artifact Registry
Now that we’ve configured our project to handle connecting the developers local environment into the project via IAP through to the database, they should be able to run the application locally while actively developing the application.

However, if they want to deploy the application into the cloud, say into a test environment, then there are more steps we need to take.  For this, we are assuming that the application is containerized and the reader is familiar with docker, so we won’t be adding the steps to create a container.

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

## Setting up Cloud Run
Now that we have our container in Artifact Registry, we can use it to create an instance of Cloud Run that will server our application.  The first thing we need to think about is networking.  Since Cloud Run is a managed service, it does not run within our VPC, yet.  That means we need to think about how to connect our Cloud Run instances to our VPC.  Fortunately, Google has provided the Serverless VPC Connector to handle this.  Let’s start by creating one.

```bash
export CONNECTOR=my-connector
gcloud compute networks vpc-access connectors create $CONNECTOR \
--region=$REGION \    
--network=$NETWORK \
--range=10.8.0.0/28 \
--min-instances=2 \
--max-instances=10 \
--machine-type=e2-micro

export DBHOST=$(gcloud sql instances describe $CLOUD_SQL --format "value(ipAddresses[0].ipAddress)")
export DBNAME=postgres
export DBUSER=postgres
export DBPASS=password

gcloud services enable run.googleapis.com

gcloud run deploy my-service --image=us-central1-docker.pkg.dev/$PROJECT_ID/$REPOSITORY/$IMAGE:$TAG --region=$REGION --ingress=all --max-instances=1 --set-env-vars=DBHOST=${DBHOST},DBNAME=${DBNAME},DBUSER=${DBUSER},DBPASS=${DBPASS} --vpc-connector=$CONNECTOR
```