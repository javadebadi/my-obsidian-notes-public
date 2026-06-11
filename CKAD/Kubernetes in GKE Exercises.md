

## Q 1.1: FastAPI app via Load Balancer

1. Create an autopilot GKE cluster
2. Once cluster creation is finished, open a cloud shell in GCP console.
3. Clone this repo that contains Kubernetes examples: https://github.com/javadebadi/kubernetes-examples
4. Navigate to `01_simple_fastapi` directory inside the cloned repo
5. Take a look at the `Dockerfile` there
6. Create a docker repo in GCP named `my-repo` (using `gcloud` or GCP console)
7. Use `gcloud builds` to use Cloud Build to build the container image from Dockerfile. Note that the correct repository artifact name and tag should be compatible with artifact registry repo, i.e. `<LOCATION>-docker.pkg.dev/<PROJECT_ID>/<REPO>/<IMAGE_NAME>:<IMAGE_TAG>`
8. After it is done, check the docker container image is in the repo
9. Create a Kubernetes Deployment manifest in `deploy.yaml` file to use this image
10. Expose the deployment using `LoadBalancer` service
11. Check for the services until external ip is avaialbe
12. Use curl to fetch data and confirm it is accessible via internet



> [!NOTE] Solution
> ```shell
> # clone the repo
> git clone https://github.com/javadebadi/kubernetes-examples
> # navgiate to directory
> cd 01_simple_fastapi
> # look at the dockerfile
> cat Dockerfile
> # create a docker repo
> gcloud artifacts repositories create my-repo --location=us-central1 --repository-format=docker \
> description="My Docker Repository"
> # use cloud build to build the docker container from Dockerfile
> gcloud builds submit --tag=us-central1-docker.pkg.dev/main-k8/my-repo/fastapi-app:1.0 .
> # check the image is in repo
> gcloud artifacts docker images list us-central1-docker.pkg.dev/main-k8/my-repo/
> # create deploy.yaml ....
> # the deploy.yaml is in below
> # Expose the deployment using svc
> kubectl expose deployment fastapi-app --port=80 --target-port=8080 --type=LoadBalancer
> # check for service until external IP is available
> kubectl get svc
> # check the access to service from internet
> curl http://<EXTERNAL_IP>:80
> ```
> 

Here is the yaml file for exercise 1:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: fastapi-app
  name: fastapi-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fastapi-app
  strategy: {}
  template:
    metadata:
      labels:
        app: fastapi-app
    spec:
      containers:
      - image: us-central1-docker.pkg.dev/main-k8/my-repo/fastapi-app:1.0
        name: fastapi-app
        resources:
          limits:
            cpu: '500m'
            memory: '128Mi'
        ports:
        - containerPort: 8080
          name: http-server
```
