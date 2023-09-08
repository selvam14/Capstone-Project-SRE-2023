![ntu_logo](./assets/ntu%20logo.png) 
# PaCE@NTU SCTP Cohort 2 - Cloud Infrastructure Engineering

## Monitoring Large Applications (Site Reliability Engineering)

**Group 3**: Soo Wei Heng | Ren Peng | Sangeetha Thirumurugan | Ng Yu Yuan | Selvam S/O P Mohan

---

## TechWise Innovations Company Profile

At TechWise Innovations, we are pioneers in the field of enterprise software solutions, with a special focus on microservices architecture. We take pride in our team of Site Reliability Engineers (SREs) who play a critical role in ensuring early detection, enhancing observability, and guaranteeing system reliability.

- [Requirements Gathering](https://docs.google.com/spreadsheets/d/11J3PLi_lxMLwQo3EYkwIn_NQOdFA3hFcKvUAmc0bSTU/edit#gid=0)
- [Jira Board](https://sctp-cloud-cohort2-group3.atlassian.net/jira/software/projects/GCP/boards/1)

---

## Monitoring and Visualization

When it comes to visualizing metrics from every component of your system, the dynamic duo of Grafana and Prometheus reign supreme:

- **Grafana**: The visualization powerhouse that transforms Prometheus data into actionable insights.
- **Prometheus**: Your go-to monitoring solution for storing critical time series data, including vital metrics.

Together, they empower you to harness the full potential of your data, ensuring that you never miss a beat in your monitoring and visualization endeavors.

---

## Solution Architecture
![solution architecture image](./assets/solution%20architecture.png)

---

## Movie App Project

The application team recently worked with an entertainment company specializing in streaming online media content to create a movie application. The movie app project used React components, router redirects, Firebase backend, and data pulled from an API with Axios.

- [GitHub Repository](https://github.com/Sule-Ss/movie-app-with-react)

The application was containerized using Docker and deployed to AWS Elastic Container Service.

![containerisation_workflow](./assets/containerisation_workflow.png)
### Cloning the Repository to Your Local Environment

![git-clone](./assets/git_clone.png)
```shell
git clone https://github.com/Sule-Ss/movie-app-with-react.git
cd movie-app-with-react
code .
```

### Create Dockerfile
![dockerfile-sample](./assets/create-dockerfile1.png)
```Dockerfile
# Use a base image
FROM node:14 as build

# Set the working directory
WORKDIR /app

# Copy package.json and package-lock.json
COPY package*.json ./

# Install dependencies
RUN npm install

# Copy the rest of the app’s files
COPY . .

# Build the React app
RUN npm run build

# Use a smaller base image for serving the app
FROM nginx:alpine

# Copy the build files to the nginx directory
COPY --from=build /app/build /usr/share/nginx/html

# Expose port 80
EXPOSE 80

# Start the nginx server
CMD ["nginx", "-g", "daemon off;"]
```

## Docker Commands
1. Open Docker Desktop and sign in.

2. Go back to your VS Code.

3. Build the Docker image with a tag (name) for the image:
```
docker build -t movie-app-image .
```
(The -t flag is used to tag or name the Docker image that is being built.)

4. Run the Docker container, mapping port 8080 on the host to port 80 in the container, and run it in detached mode:
```
docker run -p 8080:80 -d movie-app-image
```
(The -p flag is used to map or publish ports between the host and the container. For example, port 80 within the container will be mapped to port 8080 on the host machine. The -d flag is used to run a container in detached mode, which means the container runs in the background as a daemon process.)

5. Once you can deploy locally, you can proceed to deploy it on Amazon Elastic Container Service (ECS).

6. Build the Docker image again with the same tag (if needed) to ensure you have the latest image locally:
```
docker build -t movie-app-image .
```
7. The Docker image will be pushed to Amazon Elastic Container Registry (ECR). Ensure you are signed in to your AWS account via a web browser.

8. Obtain an ECR login password for the specified AWS region and log in to ECR using Docker:
```
aws ecr get-login-password --region ap-southeast-1 | docker login --username AWS --password-stdin 255945442255.dkr.ecr.ap-southeast-1.amazonaws.com
```

## Create a Repository in Amazon Elastic Container Registry (ECR)
1. Next, you need to create a repository in Amazon Elastic Container Registry (ECR) where you can store your Docker image.
2. 
![ECR1](./assets/ECR1.png)

3. Enter a unique name for your repository in the ECR console.

![ECR2](./assets/ECR2.png)

## Tag Docker Image with Amazon Elastic Container Registry
After creating the repository, you'll need to tag your Docker image with the ECR repository URL and a specific tag (e.g., "latest"). This is necessary for pushing the image to ECR.
```
docker tag movie-app-image:latest 255945442255.dkr.ecr.ap-southeast-1.amazonaws.com/movie-app-image:latest
```
(Replace "movie-app-image" with your image name and "latest" with your desired tag if different.)

## Push Image to Amazon Elastic Container Registry (ECR)
Now that your Docker image is tagged correctly, you can push it to Amazon Elastic Container Registry (ECR) using the following command:
```
docker push 255945442255.dkr.ecr.ap-southeast-1.amazonaws.com/movie-app-image:latest
```

## Terraform Configuration for Amazon Elastic Container Service (ECS)
The deployment of your container to Amazon Elastic Container Service (ECS) will be orchestrated using Terraform. You will need to create a Terraform configuration file in your Visual Studio Code (VS Code) environment. Below is the Terraform code for configuring your ECS setup:

![terraform1](./assets/terraform1.png)

```Terraform.tf
provider "aws" {
  region = "ap-southeast-1" # Modify this with your desired AWS region
}

locals {
  application_name = "movie-app-image" # Replace with your application name
}

resource "aws_ecs_task_definition" "my_task_definition" {
  family                   = local.application_name
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  execution_role_arn      = "arn:aws:iam::255945442255:role/ecsTaskExecutionRole" # Replace with your execution role ARN

  container_definitions = jsonencode([
    {
      name  = local.application_name
      image = "255945442255.dkr.ecr.ap-southeast-1.amazonaws.com/movie-app-image:latest" # Modify with your ECR image URL
      portMappings = [
        {
          containerPort = 8080
          hostPort      = 8080
          protocol      = "tcp"
        }
      ]
      essential = true
    }
  ])

  cpu    = "512"
  memory = "1024"
}

resource "aws_ecs_cluster" "my_ecs_cluster" {
  name = "${local.application_name}-cluster"
}

resource "aws_ecs_service" "my_ecs_service" {
  name            = "${local.application_name}-service"
  cluster         = aws_ecs_cluster.my_ecs_cluster.id
  task_definition = aws_ecs_task_definition.my_task_definition.arn
  launch_type     = "FARGATE"

  network_configuration {
    subnets          = ["subnet-04056e91a09a5b4bf", "subnet-bea677f6", "subnet-29ed7170"] # Modify with your subnet IDs
    assign_public_ip = true
    security_groups = ["sg-b4db57fc"] # Modify with your security group IDs
  }

  scheduling_strategy = "REPLICA"
  desired_count       = 1
  platform_version    = "LATEST"
  deployment_controller {
    type = "ECS"
  }
  deployment_maximum_percent = 200
  deployment_minimum_healthy_percent = 100
  enable_ecs_managed_tags = true
}
```

Commands used:
- terraform init
- terraform plan
- terraform apply

![terraform2](./assets/terraform2.png)

# GitHub Actions for Automation

The deployment of your application using Terraform scripts will be automated via GitHub Actions.

## Setting Up GitHub Actions Workflow

1. Inside your repository, create a `.github/workflows` directory and add a YAML file for your workflow. Name it something like `deploy-movie-app.yml`.

   ```yaml
   name: Deploy Movie App
   run-name: ${{ github.actor }} is doing CICD to deploy AWS ECS

   on:
     push:
       branches:
         - main

   jobs:
     deploy:
       name: Deploy Movie App
       runs-on: ubuntu-latest

       steps:
         - name: Checkout
           uses: actions/checkout@v3

         - name: Configure AWS credentials
           uses: aws-actions/configure-aws-credentials@v1
           with:
             aws-access-key-id: ${{secrets.AWS_ACCESS_KEY_ID}}
             aws-secret-access-key: ${{secrets.AWS_SECRET_ACCESS_KEY}}
             aws-region: ap-southeast-1

         - name: Login to Amazon ECR
           id: login-ecr
           uses: aws-actions/amazon-ecr-login@v1

         - name: Build, tag, and push image to Amazon ECR
           id: build-image
           env:
             ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
             ECR_REPOSITORY: movie-app-weiheng-test
             IMAGE_TAG: latest
           run: |
             # Build a Docker container and
             # push it to ECR so that it can
             # be deployed to ECS.
             docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
             docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
             echo "::set-output name=image::$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG"

         - name: Initialize Terraform
           run: terraform init
           working-directory: .

         - name: Plan Terraform changes
           run: terraform plan
           working-directory: .

         - name: Apply Terraform changes
           run: terraform apply -auto-approve
           working-directory: .
      ```

     Create secrets in your GitHub repository to store sensitive data like AWS access keys. Name them AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY.

## Running the GitHub Actions Workflow
To trigger the GitHub Actions workflow, use the following commands:

```
git add .
git commit -m 'Initial commit'
git push
```
Inside GitHub Actions, you can monitor the workflow's progress, debug any errors, and ensure that all steps are executed successfully.
