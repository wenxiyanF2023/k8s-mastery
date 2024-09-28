## Links
#### URLs for the Docker Hub images
[wyangjessie/sentiment-analysis-web-app](https://hub.docker.com/repository/docker/wyangjessie/sentiment-analysis-web-app/general) \
[wyangjessie/sentiment-analysis-logic](https://hub.docker.com/repository/docker/wyangjessie/sentiment-analysis-logic/general) \
[wyangjessie/sentiment-analysis-frontend](https://hub.docker.com/repository/docker/wyangjessie/sentiment-analysis-frontend/general) 

#### Recording 1
[link](https://drive.google.com/file/d/1VOkkykQ3rCi12__6WnUSHRTKdZW3ydme/view?usp=sharing)

## Step 1 Fork the [Repository](https://github.com/rinormaloku/k8s-mastery.git) and clone it
```
git clone https://github.com/wenxiyanF2023/k8s-mastery.git
cd k8s-mastery
```

## Step 2 Update Dockerfiles
#### 2.1 `sa-frontend/Dockerfile` 
- `sa-frontend/Dockerfile` original version
```
FROM nginx
COPY build /usr/share/nginx/html
```

- `sa-frontend/Dockerfile` updated version
```
FROM node:14 AS build
WORKDIR /app
COPY package.json yarn.lock ./
RUN yarn install
COPY . ./
RUN yarn build

FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

The updated Dockerfile uses a multi-stage build where the frontend code is compiled inside the Docker container itself. It starts with the node:14 base image to run Node.js commands. The app dependencies are installed (yarn install), and the frontend assets are built with yarn build. This step generates the production-ready static files inside the /app/build directory. The second stage uses nginx:alpine to serve the production-ready frontend files. The built static assets from the first stage (/app/build) are copied into directory /usr/share/nginx/html using the COPY --from=build instruction. The EXPOSE 80 and CMD instructions ensure that NGINX serves the files on port 80.


#### 2.2 `sa-web-app/Dockerfile` 
- `sa-web-app/Dockerfile` original version
```
FROM openjdk:8-jdk-alpine
# Environment Variable that defines the endpoint of sentiment-analysis python api.
ENV SA_LOGIC_API_URL http://localhost:5000
ADD target/sentiment-analysis-web-0.0.1-SNAPSHOT.jar /
EXPOSE 8080
CMD ["java", "-jar", "sentiment-analysis-web-0.0.1-SNAPSHOT.jar", "--sa.logic.api.url=${SA_LOGIC_API_URL}"]
```

- `sa-web-app/Dockerfile` updated version
```
FROM openjdk:8-jdk-alpine
FROM maven:3.8.4-jdk-8-slim AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

FROM openjdk:8-jdk-alpine
WORKDIR /app
COPY --from=build /app/target/sentiment-analysis-web-0.0.1-SNAPSHOT.jar ./app.jar
ENV SA_LOGIC_API_URL http://localhost:5000
EXPOSE 8080
CMD ["java", "-jar", "app.jar", "--sa.logic.api.url=${SA_LOGIC_API_URL}"]
```

The first stage uses a Maven container (maven:3.8.4-jdk-8-slim) to compile the Java application from source code. Dependencies are fetched (mvn dependency:go-offline), referred to [link](https://stackoverflow.com/questions/7233328/how-do-i-configure-maven-for-offline-development), and the project is packaged (mvn package -DskipTests). The target directory, which contains the built JAR file, is created inside the /app directory. The updated version builds the JAR file inside the Docker container itself by copying the pom.xml and src directory into the container, ensuring that the JAR is always freshly built. the updated Dockerfile uses the COPY --from=build instruction to copy the JAR file from the build stage into the final image. The environment variable SA_LOGIC_API_URL is still defined as before, specifying the endpoint for the sentiment analysis logic service. The CMD instruction to run the Spring Boot application using java -jar remains the same, except the JAR file is now referred to as app.jar instead of sentiment-analysis-web-0.0.1-SNAPSHOT.jar.


## Step 3 Set Up Docker Images
I want to make sure to build the images for multiple platforms, as referred to the documentation here: [Build Multi Platforms](https://docs.docker.com/build/building/multi-platform/)
Originally I built images for arm64 which was incompatible with GKE's platform (which is amd64), so I searched for the solution and figured out build the images for multi platforms fixed the problem.

#### 3.1 Set up docker buildx to use the docker-container driver
```
docker buildx create --name mybuilder --use
docker buildx inspect --bootstrap
```

#### 3.2 Build and push Docker images for each service
- Frontend (React + Nginx)
```
cd sa-frontend
docker buildx build --platform linux/amd64,linux/arm64 --no-cache -t wyangjessie/sentiment-analysis-frontend --push .
cd ..
```

- Logic (Python app)
```
cd sa-logic
docker buildx build --platform linux/amd64,linux/arm64 --no-cache -t wyangjessie/sentiment-analysis-logic --push .
cd ..
```

- Web App (Java Spring Boot)
```
cd sa-web-app
docker buildx build --platform linux/amd64,linux/arm64 --no-cache -t wyangjessie/sentiment-analysis-web-app --push .
cd ..
```


## Step 4 Set Up Google Kubernetes Engine (GKE)
#### 4.1 Create a GKE cluster
```
gcloud container clusters create sentiment-analyzer --num-nodes=1 --zone us-west1-b
```

Since I am based in SV campus, I use `us-west1-b` instead of `us-east1-a`.

#### 4.2 Get GKE cluster credentials:
```
gcloud container clusters get-credentials sentiment-analyzer --zone us-west1-b
```

## Step 5 Update YAML files under `/resource-manifests`
- Update Docker Hub username in `sa-frontend-deployment.yaml`.
```
containers:
        - image: wyangjessie/sentiment-analysis-frontend
```

- Update Docker Hub username in `sa-frontend-pod.yaml`.
```
containers:
        - image: wyangjessie/sentiment-analysis-frontend
```

- Update Docker Hub username in `sa-frontend-pod2.yaml`.
```
containers:
        - image: wyangjessie/sentiment-analysis-frontend
```

- Update Docker Hub username in `sa-logic-deployment.yaml`.
```
containers:
        - image: wyangjessie/sentiment-analysis-logic
```

- Update Docker Hub username in `sa-web-app-deployment.yaml`.
```
containers:
      - image: wyangjessie/sentiment-analysis-web-app
```

I left all services files unchanged.

## Step 6 Deploy the Application on GKE
Under directory `/resource-manifests`, run: 
```
kubectl apply -f .
```

##  Step 7 Verify Deployments
#### 7.1 Check pod status to ensure everything is running properly
```
kubectl get pods
```
The output shows all pods in the `Running` state.

#### 7.2 Check services to get the external IP addresses for the LoadBalancer services
```
kubectl get services
```


## Step 8 Access the Application
- Copy the external IP from sa-frontend-lb and visit it in the browser. I could see the React frontend served by NGINX.

- Input a sentence in the frontend, which triggered requests to the sa-web-app (Java Spring Boot) and sa-logic (Python sentiment analysis) services.


## Step 9 Clean Up
Delete the GKE cluster to avoid incurring unnecessary costs:
```
gcloud container clusters delete sentiment-analyzer --zone us-west1-b
```
