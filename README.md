# AWS-CICD
AWS Deployment Pipeline using CodeCommit, Code Deploy and Code Pipeline

## 1.Generating an SSH key authenticate with codecommit
In the local computer run  ```cd ~/.ssh``` . Run ```ssh-keygen -t rsa -b 4096`` and call the key `codecommit`, don't set any password for the key.
In the AWS console move to the IAM console. Move to Users=><your-admin-user-account> & open that user
Move to the Security Credentials  tab.
Under the AWS Codecommit section, upload an SSH key & paste in the copy of your clipboard.
Copy down the SSH key id  into your clipboard.
From your terminal, run ```nano ~/.ssh/config```  and at the top of the file add the following:
```
Host git-codecommit.*.amazonaws.com
User YOUR_SSH_KEY_ID
IdentityFile ~/.ssh/codecommit
```

Change the `KEY_ID_YOU_COPIED_ABOVE_REPLACEME`  placeholder to be the actual SSH key ID you copied above. Save the file and run a ```chmod 600 ~/.ssh/config``` to configure permissions correctly.


## 2. Creating Code Commit Repo
Move to the code commit console and create a repository.
Call it `catpipeline-codecommit-XXX`  where `XXX` is some unique numbers.

Once created, locate the connection step details and click SSH 
Locate the instructions for your specific operating system. Locate the command for Clone the repository  and copy it into your clipboard.

In your terminal, move to the folder where you want the repo stored, and run the copied command It should look like this: 
```
ssh://git-codecommit.us-east-1.amazonaws.com/v1/repos/catpipeline-codecommit-XXX
```  

diagram-export-23-04-2024-20_35_14.png

## 3.  Creating the ECR Private Repository
Create a private ECR repository with the repository name `catpipline` 
Note down the URI of the repository


## 4. Setup the Code Build Project
A fully managed continuous integration service that compiles source code, runs tests, and produces ready-to-deploy artifacts.
Eliminates the need to maintain and manage build servers, allowing developers to focus on writing code.
Allows creation of custom build environments to meet specific project requirements

Give the project name as `catpipline-build` , and then assign the source provider as *`AWS CodeCommit`*  and repository name `catpipeline-codecommit-XXX` . 
Select the branch as main  or master . 
Add the environment variables.
```
AWS_DEFAULT_REGION  = us-east-1
AWS_ACCOUNT_ID      = <AWS_ACCOUNT_ID_REPLACEME>
IMAGE_TAG           = latest
IMAGE_REPO_NAME     = <ECR_REPO_NAME_REPLACEME>
```

In the build specfication, specify the Buildspec 
For Service role  pick New Service Role  and leave the default suggested name which should be something like `codebuild-catpipeline-service-role` 
We will setup the Atrifacts later as we deal with this later

## 4. Set up IAM to give access to AWS Code Build
Open role codebuild-catpipeline-service-role , and add the inline permission
```
{
  "Statement": [
	{
	  "Action": [
		"ecr:BatchCheckLayerAvailability",
		"ecr:CompleteLayerUpload",
		"ecr:GetAuthorizationToken",
		"ecr:InitiateLayerUpload",
		"ecr:PutImage",
		"ecr:UploadLayerPart"
	  ],
	  "Resource": "*",
	  "Effect": "Allow"
	}
  ],
  "Version": "2012-10-17"
}
```

## 5. Create a buildspec.yml file 
I took a sample `buildpec.yml` file from internet ```gist.github.com/jweyrich/ee090a223f53700976cc4c2834f8c047```


Here's the `buildspec.yml`, save this at root level of the local repo.

```
version: 0.2
phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - docker build -t $IMAGE_REPO_NAME:$IMAGE_TAG .
      - docker tag $IMAGE_REPO_NAME:$IMAGE_TAG $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$IMAGE_TAG
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker image...
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$IMAGE_TAG
```

The `buildspec.yml` file is crucial for defining the sequence of operations that *`AWS CodeBuild`* should perform. It integrates tightly with other AWS services, particularly ECR for Docker image management. Each phase—pre_build, build, and post_build—serves a specific purpose in preparing, creating, and finalizing the Docker image, respectively. This setup ensures that the Docker environment is appropriately prepared before building, that the image is built and tagged correctly, and that the final product is pushed to the repository.


In the CodeBuild , you can click on Start Build , once docker container has been built it will push to ECR  that is tagged with latest.
To test this we can deploy a EC2 to check wether the docker has been successfully built properly.
Use this `CloudFormation` link to deploy an EC2 instance with docker installed accept all details, check the checkbox and create the stack. Wait for this to move into the CREATE_COMPLETE  state before continuing.
Connect to the EC2 instance and check if docker is preinstalled in the instance by using docker ps coomand.
Now run this command to authenticate to the ECR repository.
```
aws ecr get-login-password --region us-east-1 | docker login --username AWS --pasword-stdin <your-account-id>.dkr.ecr.us-east-1.amazon.com 
```

Now run the docker command to start the container

```
docker pull <ecr-image-uri>
docker images
docker run -p 80:80 <your-image-id>
```

Copy the Public IPV4 address and paste it in your browser to test whether the docker image is working or not.
Once it's working delete the CloudFormation Stack

diagram-export-23-04-2024-20_35_03.png



## 6. Setup the codePipeline
CodePipeline is a continuous delivery service that orchestrates the entire release workflow. It connects the different steps of your CI/CD pipeline, such as fetching code from CodeCommit, building with CodeBuild, and deploying with CodeDeploy


Go the CodePipeline console and click on create pipeline . Add name catpipline and to interact with any other AWS service you need to create a service role for the CodePipeline.
The service role gets automatically populated in your stead.
Add Source Stage, as AWS CodeCommit. In the Repository Name select your repo `catpipeline-codecommit-XXX`. Select Branch as master. 
In the Add Build Stage, Choose AWS CodeBuild as your build provider. Select the region us-east-1.  And give name catpipeline-build. Set Built type as Single build.
Once build is successful it should create a artifact folder in S3 bucket by default. Currently we are not generating any artifact.


Now we update our buildspec.yml to configure artifacts. Here's the latest buildspec.yml
```
version: 0.2

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
      - REPOSITORY_URI=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      - IMAGE_TAG=${COMMIT_HASH:=latest}
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...          
      - docker build -t $REPOSITORY_URI:latest .
      - docker tag $REPOSITORY_URI:latest $REPOSITORY_URI:$IMAGE_TAG    
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker image...
      - docker push $REPOSITORY_URI:latest
      - docker push $REPOSITORY_URI:$IMAGE_TAG
      - echo Writing image definitions file...
      - printf '[{"name":"%s","imageUri":"%s"}]' "$IMAGE_REPO_NAME" "$REPOSITORY_URI:$IMAGE_TAG" > imagedefinitions.json
artifacts:
  files: imagedefinitions.json
```

```REPOSITORY_URI=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME ``` : Sets a variable `REPOSITORY_URI`  with the full URI of the ECR repository, which simplifies the usage in later commands.
```COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)``` : Extracts the first 7 characters of the commit hash from the source version provided by CodeBuild, typically used for tagging images.
```IMAGE_TAG=${COMMIT_HASH:=latest}``` : Sets the `IMAGE_TAG`  variable to the commit hash if available, otherwise defaults to "latest". This allows for version-specific tagging of images.
```printf '[{"name":"%s","imageUri":"%s"}]' "$IMAGE_REPO_NAME" "$REPOSITORY_URI:$IMAGE_TAG" > imagedefinitions.json``` : Generates an imagedefinitions.json  file which is used by AWS ECS to deploy containers based on the built images.


Commit the new `buildspec.yml` to the ecr codecommit repository.
When you push the commit to the AWS repository, the CodeCommits get updated, it will generate an event which in turn will be seen by CloudWatch and it trigger a run of our CodePipeline .

## 7. Setup CodeDeploy 
*`CodeDeploy`* is a deployment service that automates application deployments to EC2 instances, on-premises servers, or AWS Lambda functions. It handles the deployment process and can implement strategies like blue/green deployments


The deploy step is going to take the Docker Image that you created during Code Build stage of the pipeline and deploy them out to ECS. We're going to have ECS running in Fargate mode frontend by the Load Balancer. The CodePipeline will run a deployment so updating the container  used by ECS Fargatewhenever the commit is made to the CodeCommit repository. 

### 7.1 Set up a Load Balancer.
Create a Internet Facing Application Load Balancer named `catpipelineALB`. 
Select the Default VPC for networking, and select all AZs
Configure a security to allow Any  IPV4 from anywhere from internet on port 80 
Create a Target group, Which will pointing to the container running inside ECS Fargate, And so it need o have target type of IP adress.  Create `catpipelineTG` as TG name. Select Protocol as HTTP and Port 80. 
Create the Load Balancer

### 7.2 Set up ECS Fargate
Create a cluster name `allthecatapps`. Select the Default VPC. 
Select the infrasturcture as `Fargate`.
Click on create.

Now we need to create a task and container definition, the way the deploy happens with ECS is that we first need to perform an installation manually. So we need to create task and container definition manually. And then we are going to create a Service and spin that service ourselves first. Once we done that we can update the service whenever we make a new Docker image.

### 7.3 Create a task definitions
Under task definition family use catpipelinedemo, and under container 1 name use `catpipeline` and  for the Image URI Copy the latest tagged Image URI form the ECR Repository. 
Set the task role and the task execution role to `ecsTaskExecutionRole`.
Click on create task definition. And Deploy the task definition manually. 

### 7.4 Create Service
Under Existing Cluster select `allthecatapps`.
Under Compute configuration, select the Launch Type option and make sure `Fargate` is selected
Under Service name, put `catpipelineservice`, and set desired tasks to `2` for HA. 
Enable Public IP Toggle.
Select Application Load Balancer, and use existing LB `catpipelineALB`, and Choose the container to load balancer as `catpipeline 80:80`.
Select the existing Listener select 80:HTTP. And use existing Target Group.
Click on to create Service.
Make sure both the task are showing last status as Running.

### 7.5 Create A deploy stage in CodePipeline
In the `catpipline` click on edit and add new stage to pipeline.
Add stage after Build and name it as `Deploy` 
Inside a Deploy stage, we are going to add an action. For Action name write `Deploy`., and for the action provider scroll down to AWS ECS. 
You will see ECS configuration. For the Input Artifact select BuildArtifact.  Select the cluster as `allthecatapps`. Select the service as `catpipelineservice`. 
In the Image definition file add `imagedefition.json` file which we created during the CodeBuild stage.

This will add to the CodePipeline, so whenever we commit anything to the our code commit repository it's going to run a build, generate a docker image, store that docker image on ECR and then it's going to execute a deploy stage which will deploy the docker image to the service that we've configured within ECS. 
All of this will happen automatically whenever we commit anything to our CodeCommit repository.

## 8. Configure Route53 to direct the traffic to the load balancer.
In the Route 53, Hosted Zone create CNAME record, with value as DNS of the ALB `catpipelineALB`.
So if you go the browser and paste the Custom domain name it will route to the ALB which will in turn route to the container run by ECS Fargate. 



diagram-export-23-04-2024-20_34_26.png
