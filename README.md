# DevOps Interview questions

## Ansible

### Usage and Features

- What have you done using Ansible?
- What are roles in Ansible?
- What are handlers and why use them?
- Explain the architecture of Ansible.
- What is a playbook in Ansible?
- Is Ansible agent-based or agentless?
- What to do if a needed module is not available in Ansible?
- What is ansible galaxy

### Comparison

- What is the difference between Ansible and Bash?
- What is the difference between Ansible and Terraform?
- Can Ansible build infrastructure, and how?

## Terraform

- What is the state file in Terraform?
- What is the difference between `terraform plan` and `terraform apply`?
- What is a backend in Terraform?
- Explain the lifecycle of a Terraform resource.
- What happens if a resource is deleted from the state file?
- What is the best practice for running Terraform using Jenkins?
- How to change a resource name in Terraform without destroying it?
- How to validate your HCL syntax?
- What tool can be used to document Terraform?
- Explain the process of creating an EKS cluster with Terraform.
- What benefits come from storing the tfstate file in an S3 bucket?
- What to do if the remote backend is deleted?
- **Terraform Workspace**

  - `terraform workspace new prod`
  - `terraform workspace new dev`
  - `terraform workspace select dev`
  - `terraform workspace list`
  - `terraform workspace show`

- **Add exist resource**

  - `terraform import aws_instance.existing_instance i-0123456789abcdef0`

- **Remove resource from statefile or ignore it without using comments**

  - `terraform state rm aws_instance.existing_instance`

- **Topics to focus on Terraform:**
  - Locking backend
  - Securing sensitive data
  - Terraform testing (Unit test)
  - Dynamic blocks & locals

## Containerization

- **Definition:** Containerization is a software deployment and runtime process that bundles an application’s code with all the files and libraries it needs to run on any infrastructure.

## Docker

### Basics

- What is the Docker engine?
- What is a Docker Architecture?
- What is a Dockerfile?
- What is the difference between `CMD` and `ENTRYPOINT` in Docker?
- Explain the difference between `ADD` and `COPY` in Docker.
- How to modify the entry point while running a container?
- Can a Windows container run on Linux?
- What are namespaces and cgroups in Docker?
- How can you reduce image build time?
- How can you reduce image size?
- How to secure docker image.
- how to take snapshot for my running container?
- docker commit vs docker tag
- what is build, ship, and run concept?

## Continuous Integration/Continuous Deployment (CI/CD)

### Concepts

- What is CI/CD?
- Describe a pipeline you created with GitHub Actions.
- How to avoid re-downloading dependencies in a pipeline?
- What is the difference between continuous delivery and continuous deployment?

### Tools

- Why use Jenkins over other automation tools?
- Explain Jenkins Master-Slave architecture.
- What is freestyle in Jenkins?
- What are SonarQube, JaCoCo, Trivy, and Apigee used for?
- What is the difference between scripted and declarative pipelines in Jenkins?

## AWS

### Core Services

- What is IAM? Explain IAM policies.
- What is the difference between an IAM role and an IAM user?
- Explain RDS and how to improve its performance.
- What is an EC2 instance and how to troubleshoot a timeout issue?
- What are the different storage services in AWS, and different storage classes?
- What is S3 versioning?
- What are EFS and FSx?
- What is the difference between a resource-based policy and an identity-based policy?
- Create a Serverless architecture
- SQS, RabitMQ, Lambda Function, Lambda@Edge, and Step Functions.
- how to make temproray S3 Access link?
- **Serverless function** – Piece of code that will be executed by a compute service, on demand.

- **Lambda trigger**– The type of event that will make a Lambda (serverless) function run. This can be another AWS service or an external input.

### Networking

- How to connect two VPCs?
- Can a VPC connect to another VPC in a different account?
- What is a NAT gateway and how does it work?
- What is Route 53?
- How to use CloudFront for caching static content from S3?

### Serverless

- How to extend the runtime of a Lambda function beyond 15 minutes?
- How does Lambda access S3 from the internet or VPC?
- How to deploy a dynamic website using serverless services?
- What is the maximum execution time for AWS Lambda?

## Linux

- How to check permissions on a file, including hidden files?
- What is a swapfile?
- What to do if the system boots into emergency mode?
- How to create and extend a partition?
- What is LVM and how to configure it?
- awk VS sed
- boot sequence
- Difference Between User Mode and Kernel Mode
- What is a kernel?
- What is the difference between GPT and MBR?
- tell me about special permession (uid + gid + sticky bit)

### Networking

- How to troubleshoot service connectivity issues?
- How to set up SSH on a machine?
- What to do if SSH fails?
- What is DNS and where is it configured?
- What is the difference between a proxy and a reverse proxy?

### Security

- What is HTTPS?
- Explain the process of acquiring a certificate.
- Describe a tool you have used for encryption.

## Git

### Usage

- Explain branching strategies.
- What is the difference between `git fetch` and `git pull`?
- What is the difference between `git rebase` and `git merge`?
- How to stash changes in Git?
- git merge conflict

## General Tech Concepts

### Scalability

- What is horizontal scaling and vertical scaling?
- What are the different types of load balancers and when should each be used?

### Miscellaneous

- What is the difference between encoding and encryption?
- What is a compiler and what is its job?
- What is the difference between a compiler and an interpreter?
- What is the difference between source packages and binary packages?
- What is swap space and its directory or file?
- What is jinja2
- **Authentication**
  - Who can access? (Username and Password, Certificate, Multi-Factor Authentication (MFA), Single Sign-On (SSO), OAuth third-party application using their Google or Facebook account)
- **Authorization**
  - What can they do? (checks what permissions the user has) like roles (Admin, Manager, Employee) (Attribute-Based Access Control RBAC) (Role-Based Access Control RBAC)

## Prometheus and Grafana

### Basics

- What is Prometheus and why is it used?
- How does Prometheus collect and store metrics?
- What are the main components of Prometheus?
- What is Grafana and how does it integrate with Prometheus?
- How do you create dashboards in Grafana?

### Advanced Usage

- How to set up alerting in Prometheus?
- How to write PromQL queries for monitoring?
- How to visualize Prometheus metrics in Grafana?
- How to secure Prometheus and Grafana?

## Azure

### Networking

- How to connect Azure with AWS using Direct Connect?

## Scenarios and Problem Solving

- You have a cluster with two RDS instances (one read-write and one read-only) connected to an application. The application experiences a 35-second delay before timing out. Propose four solutions.

- If you connect VPC1 to VPC2, and VPC2 to VPC3, will VPC1 be connected to VPC3? How to connect multiple VPCs without creating a star topology?

- You have an application for image processing that stores data in S3. Clients complain about latency. What can you do to reduce it?

- You have a stateful application where the same user needs to access the same machine, but the machine has high load. What should you do?

- You have deployed your application in Kubernetes and need to ensure it can handle high traffic. What steps would you take?

- Your Kubernetes cluster is experiencing slow deployments. What could be the reasons and how would you address them?

- Your EC2 instance is not reachable via SSH. What troubleshooting steps would you take?

- An application deployed on AWS Lambda is exceeding its memory limits. What can you do to troubleshoot and fix the issue?

# CI/CD Cycle

## Continuous Integration (CI)

### Image Scan

- Use [Trivy](https://trivy.dev/) for scanning images.

### Branch Management

- **Branch Protection Rules**
- **Branch Strategies**

### Pull Request Process

- Static Code Analysis: Use SonarQube.
- Code Coverage: Use JaCoCo.
- Build Process
- Unit Testing
- Security Scanning

### API Management

- Use Apigee for API management.

## Continuous Deployment (CD)

### Artifact Management

- Upload artifacts to Artifactory.
- Deploy artifacts in the development environment by downloading from Artifactory.

### Testing in Development

- Perform integration testing to ensure services and their integrations are functioning properly.
- Conduct smoke testing by the testing team.

### Versioning

- Use semantic versioning.

### Architecture

- Maintain both high-level and low-level architecture documentation.

## Production Deployment

### Approval

- Release manager approval is required.

### Deployment Process

- Download the artifact.
- Deploy to the production environment.
- Perform smoke testing post-deployment.

## Overview

1. **Continuous Integration (CI)**

   - Image scanning with Trivy.
   - Branch protection and strategy management.
   - Pull requests: Static code analysis (SonarQube), code coverage (JaCoCo), build, unit testing, and scanning.
   - API management with Apigee.

2. **Continuous Deployment (CD)**

   - Manage artifacts with Artifactory.
   - Deploy to development for integration and smoke testing.
   - Implement semantic versioning.
   - Maintain architecture documentation.

3. **Production Deployment**
   - Release manager approval.
   - Download and deploy artifacts.
   - Conduct smoke testing.
