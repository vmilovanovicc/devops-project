# devops-project



**Tech Stack**
- Java
- Maven
- AWS
- Docker
- Terraform
- Ansible
- Jenkins
- Git

---

## Part 1 - Java Web App with Unit Tests 

Source Code: https://github.com/codingvesna/devops-project/tree/java-web-app

App Deployed to AWS EBS: http://java-web-app.eu-west-1.elasticbeanstalk.com/

Or run on local machine:

 - Clone repo, start Tomcat Server, open port 8081 and go to browser ` http://localhost:8081/java-web-app/ `

### App Description

A Simple Web App built with Java & Maven.

**Test Cases:**

- Test Register Page - index.jsp
- Test Login page - home.jsp

### Prerequisities for App Development

- Java 17.0.1
- JWebUnit - Framework used with JUnit to test the web application.
- Apache Tomcat 9.0.56
- Apache Maven 3.8.4
- AWS Elastic Beanstalk (deployment)
- IntelliJ IDEA - IDE for app development

---

## Part 2 - Infrastructure & Configuration

### Providing infrastructure with Terraform

Scripts location: https://github.com/codingvesna/devops-project branch: `main` folder: `scripts`

### Prerequisites
1. Installed **AWS CLI**
2. Created IAM user *vesna*, generated *AWS Access Key ID & Secret Access Key*
3. Used `aws configure` command to input credentials (also can be set under the *provider* block)
4. Created **main.tf** to define the architecture


### Building Infrastructure

The **terraform** block - Terraform setup, listed required providers, in this case aws; 

The **provider** block - provider definition, *profile* - reference to the AWS credentials stored in AWS config file (access keys generated by the AWS IAM for the user *vesna*);

The **resource** block - used aws_instance as a resource type and devops_project as the resource name, so **aws_instance.devops_project** is a unique ID for the resource; 
ami refers to the *Ubuntu AMI* used in this case, created t2.micro instance with the name *DevOpsProject* 

1. `terraform init` download & install providers **REQUIRED**
2. `terraform fmt` print out the names of the modified files
3. `terraform validate` check syntax
4. `terraform apply` create the infrastructure **REQUIRED**
5. `terraform show` inspect the current state

What's defined in `main.tf`?

- create ec2 instance
- create s3 bucket for remote state
- create s3 bucket policy
- create key pair
- create security group - allow traffic on ports 22, 8080, 8081
- connect via SSH to remote instance 
- trigger ansible playbook

**Connect to EC2 via SSH --> main.tf**

1. Added variables **PATH_TO_PRIVATE_KEY** & **PATH_TO_PUBLIC_KEY**
2. Used ec2 resource **aws_key_pair** to control login access to ec2 instances &
3. file function to read the public ssh key from a file 

### Congifuration with Ansible

What's defined in `config.yml`?

- JDK, JRE
- Maven
- Jenkins
- Firewall

### Jenkins Configuration

- Maven & JDK setup - add env vars
- Install Plugins - 

---

## Demo 1 - Jenkins Freestyle Job - Build & Deploy Docker Image

Source Code: https://github.com/codingvesna/devops-project/tree/freestyle-job

Docker Image: https://hub.docker.com/repository/docker/vesnam/java-web-app

### Dockerfile
```
FROM tomcat:9
EXPOSE 8080
ADD target/*.war /usr/local/tomcat/webapps
```

### docker-compose.yml
```
version: "3.9"
services:
  app:
    image: vesnam/java-web-app
    container_name: java-web-app-freestyle
    ports:
      - '8081:8080'
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - logvolume01:/var/log
volumes:
  logvolume01: {}
```

### Freestyle Job  

- Checkout from SCM `https://github.com/codingvesna/devops-project.git` branch: `java-web-app`
- Build Project with Maven -> package `java-web-app.war` in `target` folder
- Credentials and repo name for publishing a Docker Image on DockerHub
- Use **Docker Build and Publish Plugin** - define repo name on DockerHub
- Add **docker compose build step** in Jenkins, start service / container 
- Post-build: archive the artifacts	

### Final Result

- After building image, the image is automatically pulled from DockerHub and the container is started.

- To see the web app type in your browser:` http://public_ip_address:8081/java-web-app/ `

---

## Demo 2 - Jenkins Pipeline - Build & Deploy Docker Image

Source Code: https://github.com/codingvesna/devops-project/tree/pipeline

Docker Image: https://hub.docker.com/repository/docker/vesnam/java-pipeline

## Jenkinsfile

**declarative pipeline** next step: scripted pipeline

```
pipeline {
    environment {
        registry = "vesnam/java-pipeline"
        registryCredential = 'dockerhub'
        dockerImage = ''
        container = 'java-pipeline'
    }
    agent any
    tools {
        maven 'M3'
    }
    stages {
        stage('code') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: '*/pipeline']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/codingvesna/devops-project.git']]])
            }
        }

        stage('build') {
            steps {
                sh 'mvn clean compile'
            }
        }

        stage('test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('package') {
            steps {
                script {
                  sh 'mvn package'
                }
                archiveArtifacts artifacts: 'target/*.war', followSymlinks: false
            }
        }

        stage('build docker image'){
            steps  {
                script {
                //     sh 'docker build -t vesnam/pipeline .'
                    dockerImage = docker.build registry
                }

            }

        }
        stage('deploy image') {
            steps{
                script {
                    docker.withRegistry( 'https://registry.hub.docker.com/', registryCredential ) {
                        dockerImage.push()
                    }
                }
            }
        }

        stage('run container') {
            steps {
                sh 'docker-compose up -d --build'
            }
        }

        stage('cleanup') {
            steps{
//                 sh 'docker stop $container'
//                 sh 'docker rm $container'
                sh 'docker rmi $registry'
            }
        }


    }
}
```

In this case, the pipeline is defined in `Jenkinsfile` and pulled from a GitHub repo.

 - Create credentials for DockerHub - dockerhub
 - Define variables for Docker setup
 - CI/CD Pipeline : 
    - `code` -> pull code from a `github` repo
	- `build` -> clean workspace and compile (no compile in this case)
	- `test` -> unit testin
	- `package` -> build`.war` file and archive
	- `build docker image` -> build image from file
	- `deploy image` -> image deployed to DockerHub
	- `run container` -> run container from deployed image 
	- `cleanup` -> remove unused image
	

## Dockerfile
```
FROM tomcat:9
EXPOSE 8080
ADD target/*.war /usr/local/tomcat/webapps
```

## docker-compose.yml
```
version: "3.9"
services:
   app:
      image: vesnam/java-pipeline
      container_name: ${container}
      ports:
         - '8081:8080'
      volumes:
         - /var/run/docker.sock:/var/run/docker.sock
         - logvolume03:/var/log
volumes:
   logvolume03: {}
```
### Final Result

- Check response `curl http://localhost:8081/java-web-app/`

- To see the web app type in your browser:` http://public_ip_address:8081/java-web-app/ `

---

## Demo 3 - Multi Stage Dockerfile - Jenkins Build & Deploy Docker Image


Source Code: https://github.com/codingvesna/devops-project/tree/multi-stage

Docker Image: https://hub.docker.com/repository/docker/vesnam/java-multi-stage

## Dockerfile
```
FROM maven:3.8.4-jdk-11 AS build
WORKDIR /app
COPY src src
COPY pom.xml .
RUN mvn package

FROM tomcat:9
# copying from stage
COPY --from=build /app/target/*.war /usr/local/tomcat/webapps
EXPOSE 8080

# copying from local directory
# COPY target/*.war /usr/local/tomcat/webapps
```

**EXPLAINED**

This is a `two-stage` build as there are two `FROM` statements.

The `build` (maven) stage is the base stage for the first build. This is used to build the `war` file for the app.

The `tomcat` stage is the second and final base image for the build. The `war` file generated in the `build` stage is copied over to this stage using
`COPY --from=build` syntax.

## docker-compose.yml
```
version: "3.9"
services:
  app:
    image: vesnam/java-multi-stage
    container_name: java-multi-stage
    ports:
      - '8081:8080'
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - logvolume02:/var/log
volumes:
  logvolume02: {}
```

## Final Result
- Check response `curl http://localhost:8081/java-web-app/`

- To see the web app type in your browser:` http://public_ip_address:8081/java-web-app/ `


---

## Jenkins Multiple Jobs

https://github.com/codingvesna/internship/tree/multiple-jobs

---

### Other

- EC2 Ubuntu Instance ` terraform-controller ` that is as a control node for Terraform and Ansible.
- Bash scripts `install-ansible.sh` , `install-terraform.sh`, `update.sh` - to install Terraform and Ansible on control node

