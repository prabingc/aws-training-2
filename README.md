Steps for Final project

1) Assumption is you have an instance that has following installed; we will call this instance deployment-server.
	- Docker
	- Kubernetes
	- EKSCTL
	- Git
	- Node
	- Terraform
	- Helm

	Also this instance will have to have Instance profile associated that will permit all the actions that we are about to do. Alternatively you can create a IAM user with correct permission and configure the instance using "aws configure".

	Please use the latest Amazon Linux AMI, recommended instance type is t3.small (at minimum) with 20GB EBS attached. You can use following userdata file to install above at launch.
	> curl -O https://raw.githubusercontent.com/prabingc/aws-training-2/blob/main/capstone/userdata.txt

2) Launch EKS cluster
   Connect to the deployment-server using "EC2 instance connect" or "Session Manager" or SSH client.
   ```
   mkdir ekscluster
   cd ekscluster
   curl -O https://raw.githubusercontent.com/prabingc/aws-training-2/main/capstone/cluster.yaml
   eksctl create cluster -f cluster.yaml
   ```

   This will take sometime so keep this session active and launch new session for remaining tasks.

3) Creating Docker image with app.
    ```
    mkdir ~/capstone_eventsapp
    cd ~/capstone_eventsap
    git clone https://github.com/msutton150/eventsappstart.git
   ```
4) Create docker images with apps.
    ```
	cd eventsappstart/events-api

	touch .dockerignore
	curl -o Dockerfile https://raw.githubusercontent.com/prabingc/aws-training-2/main/capstone/dockerfile_api
	docker build . -t events-api:v1.0

	cd ../events-website/
	touch .dockerignore
	curl -o Dockerfile https://raw.githubusercontent.com/prabingc/aws-training-2/main/capstone/dockerfile_website

	docker build . -t events-website:v1.0

	#
	#we will be creating 2 version of image to test our rolling updated
	#
	#edit homepage to show your name instead of place holder
	#
	vi views/layouts/default.hbs
	#
	#edit line 7 and 16 to some name; Make other edits in this page so as not to mess up HTML
	#
	docker build . -t events-website:v2.0

	#validation
	docker image

	#to test the images
	docker run -d -p 8082:8082 events-api:v1.0
	docker run -d -p 8080:8080 events-website:v1.0
    
	You should be able to test by browsing to public IP of the deployment-server using http://{ip}:8080 or http://{ip}:8082
   ```
5) Export image to ECR; you can create repo in any region but for following instruction assume you are using 'US-East-1'.

	Create 2 repo in "Amazon ECR" with events-api and events-website and copy the URI for each.
    ```
	cd ../events-api
	docker login -u AWS -p $(aws ecr get-login-password --region us-east-1) {uri for events-api}
	docker tag events-api:v1.0 {uri for events-api}:v1.0
	docker push {uri for events-api}:v1.0


	cd ../events-website/
	docker login -u AWS -p $(aws ecr get-login-password --region us-east-1) {uri for events-website}
	docker tag events-website:v1.0 {uri for events-website}:v1.0
	docker push {uri for events-website}:v1.0

	docker tag events-website:v2.0 {uri for events-websit}:v2.0
	docker push {uri for events-website}:v2.0
    ```

	Copy the URI for all 3 images that we have uploaded from EKS console
	- events-api :--> account_number.dkr.ecr.us-east-1.amazonaws.com/events-api:v1.0
	- events-website :--> account_number.dkr.ecr.us-east-1.amazonaws.com/events-website:v1.0
	- events-website :--> account_number.dkr.ecr.us-east-1.amazonaws.com/events-website:v2.0


6) Deploying apps to EKS clusters.
   ```
   mkdir ../kubernetes-config
   cd ../kubernetes-config
   curl -o api_deployment.yaml https://raw.githubusercontent.com/prabingc/aws-training-2/main/capstone/api_deployment.yaml
   		update the line 20 for image with one from previous step

   curl -o web_deployment.yaml https://raw.githubusercontent.com/prabingc/aws-training-2/main/capstone/web_deployment.yaml
   		update the line 20 for image with one from previous step

   curl -o api_service.yaml https://raw.githubusercontent.com/prabingc/aws-training-2/main/capstone/api_service.yaml
   curl -o web_service.yaml https://raw.githubusercontent.com/prabingc/aws-training-2/main/capstone/web_service.yaml


   	kubectl apply -f api_deployment.yaml
   	kubectl apply -f web_deployment.yaml
   	kubectl apply -f api_service.yaml
   	kubectl apply -f web_service.yaml

   	
   	#validation
   	kubectl get deployments
   	kubectl get pods
   	kubeclt get svc

   	Browse to http://<dns for events-web-svc>
    ```
7) Testing rolling updates
```
	Update the "web_deployment.yaml" to use v2.0 or the image

	vim web_deployment.yaml 
	#edit 20 to use v2.0 instead of v1.0 and save

	kubectl apply -f web_deployment.yaml 

	#validation
	kubectl get pods --> you will see pods creating and terminaing. 

	if you browse to the  http://{dns for events-web-svc} it will be up entire time; provided it might switch between v1 and v2.. refresh few time until you see only v2 version.
```
8) Adding database
	On Deployment server run following
```    
	- Set the default value for storage class:
		 kubectl patch storageclass gp2 -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

	- user helm to install mariadb
		helm install database-server oci://registry-1.docker.io/bitnamicharts/mariadb

	#validation
	kubectl get pods
	kubectl get svc

	helm list 
	helm status database-server 
```
9) Integrating api to use Database
	Download the new api_deployment inside kubernetes-config folder
	``` 
    curl -o api_deployment_v1.yaml https://raw.githubusercontent.com/prabingc/aws-training-2/main/capstone/api_deployment_v1.yaml 
   		update the line 20 for image with one from step 5

   	kubectl apply -f api_deployment_v1.yaml

   	validation:
   	kubectl get pods -> you should see new pod launched for database-server-mariadb-0
```