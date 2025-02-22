# Part 1 - Docker

## Step 1: Install Docker on EC2

Execute the following commands:

```bash
sudo yum update -y
sudo yum install docker -y
sudo systemctl start docker
sudo docker run hello-world
sudo systemctl enable docker
docker --version
sudo usermod -a -G docker $(whoami)
newgrp docker
```

```bash
sudo usermod -a -G docker $(whoami)
newgrp docker
```

## Step 2: Create Docker image for CloudMart

### Backend

Create folder and download source code:

```bash
mkdir -p challenge-day2/backend && cd challenge-day2/backend
wget https://tcb-public-events.s3.amazonaws.com/mdac/resources/day2/cloudmart-backend.zip
unzip cloudmart-backend.zip
```

Create `.env` file:

```bash
nano .env
```

Content of `.env`:

```
PORT=5000
AWS_REGION=us-east-1
BEDROCK_AGENT_ID=<your-bedrock-agent-id>
BEDROCK_AGENT_ALIAS_ID=<your-bedrock-agent-alias-id>
OPENAI_API_KEY=<your-openai-api-key>
OPENAI_ASSISTANT_ID=<your-openai-assistant-id>
```

Create Dockerfile:

```bash
nano Dockerfile
```

Content of Dockerfile:

```docker
FROM node:18
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 5000
CMD ["npm", "start"]
```

### Frontend

Create folder and download source code:

```bash
cd ..
mkdir frontend && cd frontend
wget https://tcb-public-events.s3.amazonaws.com/mdac/resources/day2/cloudmart-frontend.zip
unzip cloudmart-frontend.zip
```

Create Dockerfile:

```bash
nano Dockerfile
```

Content of Dockerfile:

```
FROM node:16-alpine as build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:16-alpine
WORKDIR /app
RUN npm install -g serve
COPY --from=build /app/dist /app
ENV PORT=5001
ENV NODE_ENV=production
EXPOSE 5001
CMD ["serve", "-s", ".", "-l", "5001"]

```

# Part 2 - Kubernetes

Attention: AWS Kubernetes service is not free, so when executing the hands-on below, you will be charged a few cents on your AWS account according to EKS pricing on AWS.

Remember to delete the cluster to avoid unwanted charges. Use the removal section at the end of the doc.

## Cluster Setup on AWS Elastic Kubernetes Services (EKS)

1. Create a user named `eksuser` with Admin privileges and authenticate with it. Once you run the command, follow the commands: 

IAM User<sec credentials<create access key<Use case CLI< Confirm
    
    ```yaml
    aws configure
    ```
    
2. Install the CLI tool `eksctl`
    
    ```bash
    curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
    sudo cp /tmp/eksctl /usr/bin
    eksctl version
    ```
    
3. Install the CLI tool `kubectl`
    
    ```bash
    curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.18.9/2020-11-02/bin/linux/amd64/kubectl
    chmod +x ./kubectl
    mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin
    echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc
    kubectl version --short --client
    ```
    
4. Create an EKS Cluster
    
    ```bash
    eksctl create cluster \
      --name cloudmart \
      --region us-east-1 \
      --nodegroup-name standard-workers \
      --node-type t3.medium \
      --nodes 1 \
      --with-oidc \
      --managed
    ```
    
5. Connect to the EKS cluster using the `kubectl` configuration
    
    ```bash
    aws eks update-kubeconfig --name cloudmart
    ```
    
6. Verify Cluster Connectivity
    
    ```bash
    kubectl get svc
    kubectl get nodes
    ```
    
7. Create a Role & Service Account to provide pods access to services used by the application (DynamoDB, Bedrock, etc).
    
    ```bash
    eksctl create iamserviceaccount \
      --cluster=cloudmart \
      --name=cloudmart-pod-execution-role \
      --role-name CloudMartPodExecutionRole \
      --attach-policy-arn=arn:aws:iam::aws:policy/AdministratorAccess\
      --region us-east-1 \
      --approve
    ```
    

<aside>
💡

NOTE: In the example above, Admin privileges were used to facilitate educational purposes. Always remember to follow the principle of least privilege in production environments

</aside>

## Backend Deployment on Kubernetes

### **Create an ECR Repository for the Backend and upload the Docker image to it**

```bash
Repository name: cloudmart-backend
```

### Switch to backend folder

```docker
cd ../..
cd challenge-day2/backend
```

### Follow the ECR steps to build your Docker image

Note: For the step push command ```docker build -t cloudmart-frontend .```, network error may occur. Instead use:
```docker build -t cloudmart-frontend . --network=host ```

### **Create a Kubernetes deployment file (YAML) for the Backend**

Get the image url from the ECR image repository url. 

```yaml
cd ../..
cd challenge-day2/backend
nano cloudmart-backend.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudmart-backend-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cloudmart-backend-app
  template:
    metadata:
      labels:
        app: cloudmart-backend-app
    spec:
      serviceAccountName: cloudmart-pod-execution-role
      containers:
      - name: cloudmart-backend-app
        image: public.ecr.aws/l4c0j8h9/cloudmart-backend:latest
        env:
        - name: PORT
          value: "5000"
        - name: AWS_REGION
          value: "us-east-1"
        - name: BEDROCK_AGENT_ID
          value: "xxxxxx"
        - name: BEDROCK_AGENT_ALIAS_ID
          value: "xxxx"
        - name: OPENAI_API_KEY
          value: "xxxxxx"
        - name: OPENAI_ASSISTANT_ID
          value: "xxxx"
---

apiVersion: v1
kind: Service
metadata:
  name: cloudmart-backend-app-service
spec:
  type: LoadBalancer
  selector:
    app: cloudmart-backend-app
  ports:
    - protocol: TCP
      port: 5000
      targetPort: 5000
```

