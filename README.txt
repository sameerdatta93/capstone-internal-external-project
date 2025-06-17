CICD Capstone Project:

Region: us-east-1

Description:
Sample internal and external app is used in the project where external app is exposed via Load balancer and internal app via clusterIP.
Both projects are maintained in the same Git repo for simplicity along with their Docker files. There are two branches of the repo named 'main' and 'dev'.
Initially Jenkins pipeline will determine the environment if it is Dev or Prod and will be setting the required variables like namespace, creds(maintained 2 different creds here for dev and prod), etc.
Once environment is determined, it will take a git checkout and will start building external and internal apps in parallel.
The Jenkins pipeline has steps to build and push the image to Docker and then deploy the same to EKS for both internal and external separately. 
We have used npm package manager here, and before building we are doing npm install and once it is build, we are pushing it to DockerHub.
The deployment.yaml will be feching this image and will deploy it to EKS using rolling updates.

Pre-requisite:
1. Jenkins server should be up and running.
2. Cluster is up and running.
3. Permissions are given to Docker on jenkins server using below command:
	sudo usermod -aG docker jenkins
4. Below plugins needs to be installed on jenkins:
	- AWS credentials
	- Docker
	- Docker Pipeline
	- Blue Ocean (optional: just for better viewing)
	
Steps:
1. Open Global credentials in Jenkins and add below configs:
	i. Create Username and Password entry - credentials shared separately over chat - use id as 'sd-dockerhub-creds' which adding credential. (This is for Docker)
	ii. Create two AWS credentials - use id as 'sd-aws-creds-dev' and 'sd-aws-creds-prod' respectively.
2 On jenkins homepage, click on new item and create a pipeline for prod and dev. (You can create only one and can continue with respective configurations further)
3. Once pipeline is created, go to configure and select 'Pipeline Script from SCM' under Pipeline section and then select Git under SCM, and use below git url and use the branch respectively. ('main' for prod pipeline and 'dev' for dev pipeline in Branch Specifier).
	https://github.com/sameerdatta93/capstone-internal-external-project.git
	Also select credentials which you would have created in first step. 
4. Main branch is treated as Prod and dev branch will be treated as dev.
5. Run the pipeline. It should push the image to docker and deploy to eks. Rolling deployment is used while deploying a new image to eks. 6. Deployment/service will be created in respective namespaces depending on the branch. 
7. Get the Load Balancer dns either from console or using kubectl command, and open it in browser using http and port 80. The app should be running.
