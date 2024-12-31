# Lab: Deploying and running a Spring Boot application in Azure Kubernetes

This lab teaches you how to deploy the an application to an AKS cluster.

# Steps
## Create an AKS service and Container Registry using the Azure Portal
From the Git Bash window, set a AKSCLUSTER,AKSCLUSTER,RESOURCE_GROUP environment variables to the chosen names you used in the portal
```
AKSCLUSTER=ismagi
MYACR=ismagi
RESOURCE_GROUP=ismagi_group
```
## Create a MySQL database

## Create container images and push them to Azure Container Registry
As a next step you will need to containerize your different microservice applications. You can do so by using the below starter for containerizing a spring boot application.
ou can use the below Dockerfile as a basis for your own Dockerfile.

```
FROM gcr.io/distroless/java17-debian11

ARG ARTIFACT_NAME
ARG APP_PORT

EXPOSE ${APP_PORT} 8778 9779

# The application's jar file
ARG JAR_FILE=${ARTIFACT_NAME}

# Add the application's jar to the container
ADD ${JAR_FILE} app.jar

# Run the jar file
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]

```

From the Git Bash window, set a VERSION environment variable to this version number 3.0.2.
```
VERSION=3.0.2
```
You will start by building all the microservice of the spring petclinic application. To accomplish this, run mvn clean package in the root directory of the application.
```
cd src
mvn clean package -DskipTests
```

As a next step you will need to log in to your Azure Container Registry.
```
az acr login --name $MYACR
```

Create a temporary directory for creating the docker images of each microservice and navigate into this directory.
```
mkdir -p staging-acr
cd staging-acr
```
Create a Dockerfile in this new directory and add the below content.
```
FROM gcr.io/distroless/java17-debian11
   
ARG ARTIFACT_NAME
ARG APP_PORT
   
EXPOSE ${APP_PORT} 8778 9779
   
# The application's jar file
ARG JAR_FILE=${ARTIFACT_NAME}
   
# Add the application's jar to the container
ADD ${JAR_FILE} app.jar
   
# Run the jar file
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
```
You will reuse this Dockerfile for each microservice, each time using specific arguments.
Copy all the compiled jar files to this directory.
```
cp ../spring-petclinic-api-gateway/target/spring-petclinic-api-gateway-$VERSION.jar spring-petclinic-api-gateway-$VERSION.jar
cp ../spring-petclinic-admin-server/target/spring-petclinic-admin-server-$VERSION.jar spring-petclinic-admin-server-$VERSION.jar
cp ../spring-petclinic-customers-service/target/spring-petclinic-customers-service-$VERSION.jar spring-petclinic-customers-service-$VERSION.jar
cp ../spring-petclinic-visits-service/target/spring-petclinic-visits-service-$VERSION.jar spring-petclinic-visits-service-$VERSION.jar
cp ../spring-petclinic-vets-service/target/spring-petclinic-vets-service-$VERSION.jar spring-petclinic-vets-service-$VERSION.jar
cp ../spring-petclinic-config-server/target/spring-petclinic-config-server-$VERSION.jar spring-petclinic-config-server-$VERSION.jar
cp ../spring-petclinic-discovery-server/target/spring-petclinic-discovery-server-$VERSION.jar spring-petclinic-discovery-server-$VERSION.jar
```

Run an "docker build" and "docker push" command to build the container image for the api-gateway and push it to your Azure Container Registry.
```
docker build -t $MYACR.azurecr.io/spring-petclinic-api-gateway:$VERSION \
    --build-arg ARTIFACT_NAME=spring-petclinic-api-gateway-$VERSION.jar \
    --build-arg APP_PORT=8080 \
    .

docker image list
   
docker push $MYACR.azurecr.io/spring-petclinic-api-gateway:$VERSION
```

You are indicating here that the image should be tagged as <your-registry-name>.azurecr.io/spring-petclinic-api-gateway:3.0.2. The ARTIFACT_NAME is the jar file you want to copy into the container image which is needed to run the application. Each microservice also needs a port it will be exposed on. For the api-gateway this is port 8080.

Now execute the same steps for the other services.
```
docker build -t $MYACR.azurecr.io/spring-petclinic-admin-server:$VERSION \
    --build-arg ARTIFACT_NAME=spring-petclinic-admin-server-$VERSION.jar \
    --build-arg APP_PORT=8080 \
    .
   
docker push $MYACR.azurecr.io/spring-petclinic-admin-server:$VERSION

docker build -t $MYACR.azurecr.io/spring-petclinic-customers-service:$VERSION \
    --build-arg ARTIFACT_NAME=spring-petclinic-customers-service-$VERSION.jar \
    --build-arg APP_PORT=8080 \
    .
   
docker push $MYACR.azurecr.io/spring-petclinic-customers-service:$VERSION

docker build -t $MYACR.azurecr.io/spring-petclinic-visits-service:$VERSION \
    --build-arg ARTIFACT_NAME=spring-petclinic-visits-service-$VERSION.jar \
    --build-arg APP_PORT=8080 \
    .
   
docker push $MYACR.azurecr.io/spring-petclinic-visits-service:$VERSION

docker build -t $MYACR.azurecr.io/spring-petclinic-vets-service:$VERSION \
    --build-arg ARTIFACT_NAME=spring-petclinic-vets-service-$VERSION.jar \
    --build-arg APP_PORT=8080 \
    .
docker push $MYACR.azurecr.io/spring-petclinic-vets-service:$VERSION

docker build -t $MYACR.azurecr.io/spring-petclinic-config-server:$VERSION \
    --build-arg ARTIFACT_NAME=spring-petclinic-config-server-$VERSION.jar \
    --build-arg APP_PORT=8888 \
    .
   
docker push $MYACR.azurecr.io/spring-petclinic-config-server:$VERSION

docker build -t $MYACR.azurecr.io/spring-petclinic-discovery-server:$VERSION \
    --build-arg ARTIFACT_NAME=spring-petclinic-discovery-server-$VERSION.jar \
    --build-arg APP_PORT=8761 \
    .
   
docker push $MYACR.azurecr.io/spring-petclinic-discovery-server:$VERSION
```

## Deploy the microservices of the Spring Petclinic app to the AKS cluster
You now have an AKS cluster deployed and a container registry holding all the microservices docker images. As a next step you will deploy these images from the container registry to your AKS cluster.

To do this you will first need to login to your container registry to push the container images and connect to your AKS cluster to be able to issue kubectl statements. Next you will need to provide YAML deployments and execute these on your cluster.
Deploy all the microservices to the cluster in a dedicated spring-petclinic namespace and make sure they can connect to the database.

Make sure the api-gateway and admin-server microservices have public IP addresses available to them. Also make sure the spring-cloud-config-server is discoverable at the config-server DNS name within the cluster.

You also want to make sure the config-server is deployed first and up and running, followed by the discovery-server. Only once these 2 are properly up and running, start deploying the other microservices. The order of the other services doesn’t matter and can be done in any order.

You can use the below YAML file as a basis for your deployments on AKS:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: #appname#
  name: #appname#
spec:
  replicas: 1
  selector:
    matchLabels:
      app: #appname#
  template:
    metadata:
      labels:
        app: #appname#
    spec:
      containers:
      - image: #image#
        name: #appname#
        env:
        - name: "CONFIG_SERVER_URL"
          valueFrom:
            configMapKeyRef:
              name: config-server
              key: CONFIG_SERVER_URL
        imagePullPolicy: Always
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /actuator/health
            port: #appport#
            scheme: HTTP
          initialDelaySeconds: 180
          successThreshold: 1
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /actuator/health
            port: #appport#
            scheme: HTTP
          initialDelaySeconds: 10
          successThreshold: 1
        ports:
        - containerPort: #appport#
          name: http
          protocol: TCP
        - containerPort: 9779
          name: prometheus
          protocol: TCP
        - containerPort: 8778
          name: jolokia
          protocol: TCP
        securityContext:
          privileged: false


---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: #appname#
  name: #service_name#
spec:
  ports:
  - port: #appport#
    protocol: TCP
    targetPort: #appport#
  selector:
    app: #appname#
  type: #service_type#
```

As a first step, make sure you can log in to the AKS cluster. The az aks get-credentials command will populate your kubeconfig file.
```
az aks get-credentials -n $AKSCLUSTER -g $RESOURCE_GROUP
```

You will now create a namespace in the cluster for your spring petclinic microservices.
```
NAMESPACE=spring-petclinic
kubectl create ns $NAMESPACE
```

On your local filesystem create a kubernetes directory in the root of the project and navigate to it.
```
cd ..
mkdir kubernetes
cd kubernetes
```

In this directory create a file named config-map.yml with the below code and save the file.
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-server
data:
  # property-like keys; each key maps to a simple value
  CONFIG_SERVER_URL: "http://config-server:8888"
```

This YAML file describes a kubernetes ConfigMap object. In the ConfigMap you define 1 configuration setting CONFIG_SERVER_URL. This will be picked up by the pods you’ll deploy in one of the next steps. Spring boot will use these values to find the config server in your cluster.

In the folder where you created the config-map.yml file, issue the below bash statement to apply the ConfigMap in the AKS cluster.
```
kubectl create -f config-map.yml --namespace spring-petclinic
```

In the kubernetes folder copy the contents of the spring-petclinic-api-gateway.yml file.
```
curl -o spring-petclinic-api-gateway.yml https://raw.githubusercontent.com/Azure-Samples/java-microservices-aks-lab/main/docs/02_lab_migrate/spring-petclinic-api-gateway.yml
```

This file uses a replacement value for the image. This will be different for your specific container registry. Use sed to replace this value.
```
IMAGE=${MYACR}.azurecr.io/spring-petclinic-api-gateway:$VERSION
sed -i "s|#image#|$IMAGE|g" spring-petclinic-api-gateway.yml
```

In the same way, copy the contents of the spring-petclinic-admin-server.yml file.
```
curl -o spring-petclinic-admin-server.yml https://raw.githubusercontent.com/Azure-Samples/java-microservices-aks-lab/main/docs/02_lab_migrate/spring-petclinic-admin-server.yml

IMAGE=${MYACR}.azurecr.io/spring-petclinic-admin-server:$VERSION
sed -i "s|#image#|$IMAGE|g" spring-petclinic-admin-server.yml

curl -o spring-petclinic-customers-service.yml https://raw.githubusercontent.com/Azure-Samples/java-microservices-aks-lab/main/docs/02_lab_migrate/spring-petclinic-customers-service.yml

IMAGE=${MYACR}.azurecr.io/spring-petclinic-customers-service:$VERSION
sed -i "s|#image#|$IMAGE|g" spring-petclinic-customers-service.yml

curl -o spring-petclinic-customers-service.yml https://raw.githubusercontent.com/Azure-Samples/java-microservices-aks-lab/main/docs/02_lab_migrate/spring-petclinic-customers-service.yml

IMAGE=${MYACR}.azurecr.io/spring-petclinic-customers-service:$VERSION
sed -i "s|#image#|$IMAGE|g" spring-petclinic-customers-service.yml

curl -o spring-petclinic-visits-service.yml https://raw.githubusercontent.com/Azure-Samples/java-microservices-aks-lab/main/docs/02_lab_migrate/spring-petclinic-visits-service.yml

IMAGE=${MYACR}.azurecr.io/spring-petclinic-visits-service:$VERSION
sed -i "s|#image#|$IMAGE|g" spring-petclinic-visits-service.yml

curl -o spring-petclinic-vets-service.yml https://raw.githubusercontent.com/Azure-Samples/java-microservices-aks-lab/main/docs/02_lab_migrate/spring-petclinic-vets-service.yml

IMAGE=${MYACR}.azurecr.io/spring-petclinic-vets-service:$VERSION
sed -i "s|#image#|$IMAGE|g" spring-petclinic-vets-service.yml

curl -o spring-petclinic-config-server.yml https://raw.githubusercontent.com/Azure-Samples/java-microservices-aks-lab/main/docs/02_lab_migrate/spring-petclinic-config-server.yml

IMAGE=${MYACR}.azurecr.io/spring-petclinic-config-server:$VERSION
sed -i "s|#image#|$IMAGE|g" spring-petclinic-config-server.yml

curl -o spring-petclinic-discovery-server.yml https://raw.githubusercontent.com/Azure-Samples/java-microservices-aks-lab/main/docs/02_lab_migrate/spring-petclinic-discovery-server.yml

IMAGE=${MYACR}.azurecr.io/spring-petclinic-discovery-server:$VERSION
sed -i "s|#image#|$IMAGE|g" spring-petclinic-discovery-server.yml

```

You can now use each of the yaml files to deploy your microservices to your AKS cluster. You can set the default namespace you will be using as a first step, so you don’t need to repeat the namespace you want to deploy in in each kubectl apply statement. Start with deploying the config-server and wait for it to be properly up and running.
```
kubectl config set-context --current --namespace=$NAMESPACE
   
kubectl apply -f spring-petclinic-config-server.yml 
kubectl get pods -w
```

Once the config-server is properly up and running, escape out of the pod watch statement with Ctrl+Q. Now in the same way deploy the discovery-server.
```
kubectl apply -f spring-petclinic-discovery-server.yml
kubectl get pods -w
```

Once the discovery-server is properly up and running, escape out of the pod watch statement with Ctrl+Q. Now in the same way deploy the rest of the microservices.
```
kubectl apply -f spring-petclinic-customers-service.yml
kubectl apply -f spring-petclinic-visits-service.yml
kubectl apply -f spring-petclinic-vets-service.yml
kubectl apply -f spring-petclinic-api-gateway.yml
kubectl apply -f spring-petclinic-admin-server.yml
```

You can now double check whether all pods got created correctly.
```
kubectl get pods
```
##Test the application through the publicly available endpoint

Now that you have deployed each of the microservices, you will test them out to see if they were deployed correctly.
You configured both the api-gateway and the admin-server with a loadbalancer. Double check whether public IP’s were created for them.
```
kubectl get services -n spring-petclinic
```
Both the admin-server and the api-gateway should have an external IP.

Copy the external IP of the api-gateway and use a browser window to connect to port 8080. It should show you information on the pets and visits coming from your database.
You now have the Spring Petclinic application running properly on the AKS cluster.
