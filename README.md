# MLOps: Jenkins & Bring-Your-Own-Algorithm

In this workshop, we will focus on building a pipeline to train and deploy a model using Amazon SageMaker training instances and hosting on persistent Sagemaker endpoint instance(s).  The orchestration of  the training and deployment will be done through [Jenkins](https://www.jenkins.io/).  


Applying DevOps practices to Machine Learning (ML) workloads is a fundamental practice to ensure machine learning workloads are deployed using a consistent methodology with traceability, consistency, governance and quality gates. MLOps involves applying practices such as CI/CD,Continuous Monitoring, and Feedback Loops to the Machine Learning Development Lifecycle. 

This workshop will focus primarily on setting up a base deployment pipeline in Jenkins.  The expectation would be to continue to iterate on a base pipeline incorporating more quality checks and pipeline features including the consideration for a data worflow pipeline.  One component that we often see in a pipeline or between the hand-off of model build and model deploy activities is a model registry.   That is not in scope for this base pipeline but should be considered as you iterate on your pipeline.  

 There is no one-size-fits-all model for creating a pipeline; however, the same general concepts explored in this lab can be applied and extended across various services or tooling to meet the same end result.  



-----

## Workshop Contents

For this portion of the workshop, we will be building the following pipeline:  

![BYO Workshop Setup](images/Jenkins-Pipeline.png)

The stages above are broken out into:

**Checkout:** Code is checked out from this repository 

**BuildPushContainer:** Container image is built using code pulled from Git and pushed to ECR

**TrainModel:** Using the built container image in ECR, training data, and configuration code this step starts the training job using Amazon SageMaker Training Instances

**TrainStatus:** Check if training was successful. Upon completion, packaged model artifact (model.tar.gz) will be PUT to S3 model artifact bucket.

**DeployToTest:** Package model, configure model test endpoint, and deploy test endpoint using Amazon SageMaker Hosting Instances

**SmokeTest:** Ensure we are able run prediction requests against deployed endpoint

**DeployToProd:** Package model, configure model production endpoint, and deploy production endpoint using AWS CloudFormation deploying to Amazon SageMaker Hosting Instances 

*Note: For this workshop, we are deploying to 2 environments (Test/Production). In reality, this number will vary depending on your environment and the workload.  For simplicity in the lab, we deploy both to a single AWS account.  However, AWS best practices for governance and workload isolation typically include deploy across accounts.*

The architecture used for the pipeline steps is shown below: 

![BYO Workshop](images/mlops-jenkins-BaseLab.png)

-------
## Prerequisite

1) AWS Account & Administrator Access
2) Please use North Virginia, **us-east-1** for this workshop
3) This workshop assumes you have an existing installation of Jenkins or Cloudbees if you are running on your own. If running along with AWS as part of an event, you can use the cloudformation to setup a jenkins server


------

## Lab Overview

This lab is based on the [scikit_bring_your_own](https://github.com/awslabs/amazon-sagemaker-examples/blob/master/advanced_functionality/scikit_bring_your_own/scikit_bring_your_own.ipynb) SageMaker example notebook.  Please reference the notebook for detailed description on the use case as well as the custom code for training, inference, and creating the docker container for use with SageMaker.  

Although Amazon SageMaker now has native integrations for [Scikit](https://aws.amazon.com/blogs/machine-learning/amazon-sagemaker-adds-scikit-learn-support/), this notebook example does not rely on those integrations so is representative of any BYO* use case.  

Using the same code (with some minor modifications) from the SageMaker example notebook, we will utilize this GitHub repository as our source repository and our SCM into our Jenkins pipeline. 

* *Optional: You can also choose to fork this repository if you want to modify code as part of your pipeline experiments. If you fork this repository, please ensure you update github configuration references within the lab* 

This lab will walk you through the steps required to setup a base pipeline responsible for orchestration of workflow to build and deploy custom ML models to target environments

For this lab, we will perform a lot of steps manually in AWS that would typically be performed through Infrastructure-as-Code using a service like [CloudFormation](https://aws.amazon.com/cloudformation/). For the purposes of the lab, we will go step-by-step on the console so attendees become more familiar with expected inputs and artifacts part of a ML deployment pipeline.  

## **IMPORTANT:  There are steps in the lab that assume you are using N.Virginia (us-east-1).  Please use us-east-1 for this workshop** 

---

# Workshop Setup & Preparation with Cloudformation (Including Jenkins Server)

The steps below are included for the setup of AWS resources we will be using in the lab environment.

## Step 1: Create a standalone jenkins server on EC2

We will create jenkins server from a pre-baked Machine Image. This contains all the required plugins and softwares (Docker,Git) preinstalled.

 1) Download the cloudformation template for [Jenkins Server](https://github.com/chethancmk/mlops-sagemaker-jenkins-byo/blob/master/deploy/cfn-jenkins-server.yml) to your  local machine.
 2) From your AWS Account, go to **Services**-->**CloudFormation** in the region N Virginia (us-east-1) 
 3) Select create stack with new resources on the top right hand menu
 4) Select Upload a template file and Click the button to 'Choose File' that you downloaded into your local machine . Click Next
 5) Enter Stack name as 'Jenkins-Server' . Click Next
 6) Do not change anything here . Click Next
 7) Scroll to the bottom. Select the checkbox "I acknowledge that AWS CloudFormation might create IAM resources." . Click Create Stack
 8) Wait for the stack to be created and then in the output tab get the IP Address for the server
 9) The Jenkins server is accessible at the link http://<Your Jenkins IP>:8080
 10) Ask your lab instructor for the default userid and password
 
## Step 2: Create ECR Repo, S3 Buckets and Lambda Function
 
We will next create the required resources for the Lab which includes a ECR Repository for storing the custom build docker image used for training and inference. S3 Buckets to store the Model Artifact and Training Data. Lambda Function to Evaluate/Test the Model Endpoint along with required Roles.

 1) Download the cloudformation template for [MLOps w/ Jenkins Resources](https://github.com/chethancmk/mlops-sagemaker-jenkins-byo/blob/master/deploy  /cfn_mlops_jenkins_resources.yml) to your local machine.
 2) From your AWS Account, go to **Services**-->**CloudFormation** in the region N Virginia (us-east-1).
 3) Select create stack with new resources on the top right hand menu
 4) Select Upload a template file and Click the button to 'Choose File' that you downloaded into your local machine . Click Next
 5) Enter Stack name as 'MLOps-Jenkins'. Enter Unique bucket names for storing Model Artifact (*yourinitials*-jenkins-scikitbyo-modelartifact
 ) and Model Data (*yourinitials*-jenkins-scikitbyo-data). Click Next
 6) Do not change anything here . Click Next
 7) Scroll to the bottom. Select the checkbox "I acknowledge that AWS CloudFormation might create IAM resources." . Click Create Stack 
 8) Wait for the stack to be created and then in the output tab the parameters required for configuring the pipeline will be available. The values include (ECR Repository, Sagemaker Execution Role, S3 Buckets for Data and Model Artifact) Copy the values for reference later
 
 ![CFN Output](images/Jenkins_resources_output.PNG)
 
## Step 3: Copy Testing and Train Data to the training bucket

 1) Download the train and test csv files to you local machine from [Data Files](https://github.com/chethancmk/mlops-sagemaker-jenkins-byo/tree/master/data) 
 2) From your AWS Account, go to **Services**-->**S3**
 3) Click  the data bucket created previously ({initials}-jenkins-scikitbyo-data)  
 4) Click on Upload files and select the local train and test files into the bucket. This will be used in the pipeline.

Jump to [Build the Jenkins Pipeline](https://github.com/chethancmk/mlops-sagemaker-jenkins-byo/blob/master/README.md#build-the-jenkins-pipeline) after this.
 
---

# Workshop Setup & Preparation - Manual
 
Only Follow these Steps if you are running on your own and ** Cloudformation is not used** . Otherwise jump to [Build the Jenkins Pipeline](https://github.com/chethancmk/mlops-sagemaker-jenkins-byo/blob/master/README.md#build-the-jenkins-pipeline)

## Step 1: Create Elastic Container Registry (ECR)

In this workshop, we are using Jenkins as the Docker build server; however, you can also choose to use a secondary build environment such as [AWS Code Build](https://aws.amazon.com/codebuild) as a managed build environment that also integrates with other orchestration tools such as Jenkins. Below we are creating an Elastic Container Registry (ECR) where we will push our built docker images. This images we push to this registry will become input container images used for our SageMaker training and deployment.   

1) Login to the AWS Account provided
2) Verify you are in **us-east-1/N.Virginia**
3) Go to **Services** -> Select **Elastic Container Registry**
4) Select **Create repository**
   * For **Repository name**: Enter a name (ex. jenkins-byo-scikit-janedoe)
   * Toggle the **Image scan settings** to **Enabled**  (*This will allow for automatic vulnerability scans against images we push to this repository*)

![BYO Workshop Setup](images/ECR-Repo.png)

5) Click **Create repository**
6) Confirm your repository is in the list of repositories
7) You will need your repository URI in a later step so copy that URI for later. 

## Step 2: Create Model Artifact Repository

Create the S3 bucket that we will use as our packaged model artifact repository.  Once our SageMaker training job completes successfully, a new deployable model artifact will be PUT to this bucket. In this lab, we version our artifacts using the consistent naming of the build pipeline ID.  However, you can optionally enable versioning on the S3 bucket as well.  

1) From your AWS Account, go to **Services**-->**S3**
2) Click  **Create bucket**
3) Under **Create Bucket / General Configuration**:
 
   * **Bucket name:** *yourinitials*-jenkins-scikitbyo-modelartifact     
      *Example: jd-jenkins-scikitbyo-modelartifact*
   * Leave all other settings as default, click **Create bucket**

## Step 3: Create Bucket for Training/Testing Data. Copy files to the Bucket

Create the S3 bucket that we will use for storing our test and train data.  The training job will use the train.csv from the bucket to train the model. The testing lambda function will use the test.csv to smoke test the end point

1) From your AWS Account, go to **Services**-->**S3**
2) Click  **Create bucket**
3) Under **Create Bucket / General Configuration**:
 
   * **Bucket name:** *yourinitials*-jenkins-scikitbyo-data     
      *Example: jd-jenkins-scikitbyo-data*
   * Leave all other settings as default, click **Create bucket**
   
 4) Copy the train, test files from the data directory to the bucket. [Data Files](https://github.com/chethancmk/mlops-sagemaker-jenkins-byo/tree/master/data)

## Step 4: Create SageMaker Execution Role

Create the IAM Role we will use for executing SageMaker calls from our Jenkins pipeline  

1) From your AWS Account, go to **Services**-->**IAM**
2) Select **Roles** from the menu on the left, then click **Create role**
3) Select **AWS service**, then **SageMaker**
4) Click **Next: Permissions**
5) Click **Next: Tags**
6) Click **Next: Review**
7) Under **Review**:
  * **Role name:** MLOps-Jenkins-SageMaker-ExecutionRole-*YourInitials*
8) Click **Create role**
9) You will receive a notice the role has been created, click on the link to the role and make sure grab the arn for the role we just created as we will use it later. 

![BYO Workshop Setup](images/SageMaker-IAM.png)

10) We want to ensure we have access to S3 as well, so under the **Permissions** tab, click **Attach policies**
11) Type **S3** in the search bar and click the box next to 'AmazonS3FullAccess', click **Attach policy**
*Note: In a real world scenario, we would want to limit these privileges significantly to only the privileges needed.  This is only done for simplicity in the workshop.*

## Step 5: Create the Lambda Helper function

In this step, we'll create the Lambda Helper Functions that to facilitate the integration of SageMaker training and deployment into a Jenkins pipeline:

1) Go to **Services** -> Select **Lambda**
2) Create a function MLOps-InvokeEndpoint-scikitbyo with Python 3.8 as the run time and Permissions with TeamRole . 
3) Upload the lambda zip code from the /lambdas folder of this repo  [Github Link](https://github.com/chethancmk/mlops-sagemaker-jenkins-byo/tree/master/lambda)


The description of each Lambda function is included below:
 
-	**MLOps-InvokeEndpoint-scikitbyo:** This Lambda function is triggered during our "TestEvaluate" stage in the pipeline where we are checking to ensure that our inference code is in sync with our training code by running a few sample requests for prediction to the deployed test endpoint.  We are running this step before committing our newly trained model to a higher level environment.  

---

#  Build the Jenkins Pipeline

In this step, we will create a new pipeline that we'll use to:

   1) Pull updated training/inference code
   2) Create a docker image
   3) Push docker image to our ECR
   4) Train our model using Amazon SageMaker Training Instances    
   5) Deploy our model to a Test endpoint using CloudFormaton to deploy to Amazon SageMaker Hosting Instances 
   6) Execute a smoke test to ensure our training and inference code are in sync
   7) Deploy our model to a Production endpoint using CloudFormaton to deploy to Amazon SageMaker Hosting Instances

## Step 5: Configure the Jenkins Pipeline

1) Login Jenkins portal using the information provided by your instructors
2) From the left menu, choose **New Item** 
3) **Enter Item Name:** sagemaker-byo-pipeline-*yourinitials* 

    *Example: sagemaker-byo-pipeline-jd

4) Choose **Pipeline**, click **OK**

![BYO Workshop Setup](images/Jenkins-NewItem.png)

5) Under **General** tab, complete the following: 

* **Description:** Enter a description for the pipeline as "Basic Jenkins End to End Pipeline using Amazon Sagemaker Training and Hosting"

* Select **GitHub project** & Enter the following project url:: https://github.com/chethancmk/mlops-sagemaker-jenkins-byo


![BYO Workshop Setup](images/Jenkins-NewItem-1.png)


   * Scroll Down & Select **This project is parameterized**:
      -  select **Add Parameter** --> **String Parameter**
      -  **Name**: ECRURI
      -  **Default Value**: *Enter the ECR repository created above*

![BYO Workshop Setup](images/Jenkins-NewItem-2.png)


 * We're going to use the same process above to create the other configurable parameters we will use as input into our pipeline. Select **Add Parameter** each time to continue to add the additional parameters below: 

    * Parameter #2: SageMaker Execution Role 
       - **Type:** String
       - **Name:** SAGEMAKER_EXECUTION_ROLE_TEST
       - **Default Value:** *Enter the ARN of the role we created above*

   * Parameter #3: Model Artifact Bucket 
       - **Type:** String
       - **Name:** S3_MODEL_ARTIFACTS
       - **Default Value:** *yourinitials*-jenkins-scikitbyo-modelartifact
 
    * Parameter #4: S3 Bucket w/ Training and Test Data
       - **Type:** String
       - **Name:** S3_DATA_BUCKET
       - **Default Value:** *yourinitials*-jenkins-scikitbyo-data


5) Scroll down --> Under **Build Triggers** tab: 

  * Select **GitHub hook trigger for GITScm polling**

6) Scroll down --> Under **Pipeline** tab: 

   * Select **Pipeline script from SCM**
   * **SCM:** Select **Git** from dropdown
   * **Repository URL:** https://github.com/chethancmk/mlops-sagemaker-jenkins-byo   
   * **Branches to Build:** */master
   * **Credentials:** -none- *We are pulling from a public repo* 
   * **Script Path:** *Ensure 'Jenkinsfile' is populated*

7) Leave all other values default and click **Save**

We are now ready to trigger our pipeline.  But first, let's explore the purpose of Jenkinsfile.  Jenkins allows for many types of pipelines such as free style projects, pipelines (declarative / scripted), and external jobs.  In our original setup we selected [Pipeline](https://www.jenkins.io/doc/book/pipeline/getting-started/).  Our Git Repository also has a file named *Jenksfile* which contains the scripted flow of stages and steps for our pipeline in Jenkin's declarative pipeline language.  This is often referred to as pipeline-as-code as we can programmatically modify our pipelines and also ensure it remains under source control.

---

## Step 6: Trigger Pipeline Executions

In this step, we will execute the pipeline manually and then we will later demonstrate executing the pipeline automatically via a push to our connected Git Repository.  

Jenkins allows for the ability to create additional pipeline triggers and embed step logic for more sophisticated pipelines.Another common trigger would be for retraining based on a schedule, data drift, or a PUT of new training data to S3. 

1. Let's trigger the first execution of our pipeline. While you're in the Jenkins Portal, select the pipeline you created above:  

![BYO Workshop Setup](images/Jenkins-Pipeline-Trigger.png)

2. Select **Build with Parameters** from the left menu:

![BYO Workshop Setup](images/Jenkins-Pipeline-Trigger-2.png)

3. You'll see all of the parameters we setup on initial configuration, you could change these values prior to a new build but we are going to leave all of our defaults, then click **Build** 

4. You can monitor the progress of the build through the dashboard as well as the stages/steps being setup and executed.  An example of the initial build in progress below: 

![BYO Workshop Setup](images/Jenkins-Pipeline-Trigger-3.png)


*Note: The progress bar in the lower left under Build History will turn a solid color when the build is complete. 

5. Once your pipeline completes the **BuildPushContainer** stage, you can go view your new training/inference image in the repository we setup: Go To [ECR](https://console.aws.amazon.com/ecr/repositories). Because we turned on vulnerability scanning, you can also see if your image has any vulnerabilities.  This would be a good place to put in a quality gate, stopping the build until the vulnerabilities are addressed: 

![BYO Workshop Setup](images/ECR-Repo-Push.png)


6. When your pipeline reaches the **TrainModel** stage, you can checkout more details about your model training within SageMaker.  While we use Jenkins to orchestrate the kickoff of our training, we are still utilizing Amazon SageMaker training features &  instances for this step.  Go To [Amazon SageMaker Training Jobs](https://console.aws.amazon.com/sagemaker/home?region=us-east-1#/jobs).  You can click on your job and review the details of your training job as well as check out the system monitoring metrics.  
7. When the pipeline has completed the **TrainStatus** stage, the model has been trained and you will be able to find your deployable model artifact in the S3 bucket we created earlier.  Go To [S3](https://s3.console.aws.amazon.com/s3/home?region=us-east-1) and find your bucket to view your model artifact: *yourinitials*-jenkins-scitkitbyo-modelartifact
8. When the pipeline has completed, the model that was trained once is deployed to test, run through a smoke test, and deploy to production.  Go to [Amazon SageMaker Endpoints](https://console.aws.amazon.com/sagemaker/home?region=us-east-1#/endpoints) to check out your endpoints.  

![BYO Workshop Setup](images/SageMaker-Endpoints.png)



**CONGRATULATIONS!** 

You've setup your base Jenkins pipeline for building your custom machine learning containers to train and host on Amazon SageMaker.  You can continue to iterate and add in more functionality including items such as: 

 * A/B Testing 
 * Model Registry
 * Data Pre-Processing 
 * Feature Store Integration
 * Additional Quality Gates
 * Retraining strategy
 * Include more sophisticated logic in the pipeline such as [Inference Pipeline](https://docs.aws.amazon.com/sagemaker/latest/dg/inference-pipelines.html), [Multi-Model Endpoint](https://docs.aws.amazon.com/sagemaker/latest/dg/multi-model-endpoints.html)



# Invoking the Endpoint through Public API (Bonus Lab)

This is a bonus step to expose the Inference Endpoint to the external applications using a API Gateway and Lambda. The private endpoint created so far is not public and requires a wrapper for making it available to the external world. Here we create a lambda function and API gateway which takes user input through API tools like postman and returns the inference result. 

![Public API Endpoint](images/apigw-lambda-endpoint.jpg)


## Step 1: Create a Cloudformation Stack with API Gateway and Lambda

In this step we will load a cloudformation template to create the resources required to expose the endpoint externally


1) Download the Cloudformation template locally [API Gateway+Lambda](https://github.com/chethancmk/mlops-sagemaker-jenkins-byo/blob/master/deploy/cfn_mlops_apigw.yml)
2) Login to the AWS Account provided
3) Verify you are in **us-east-1/N.Virginia**
4) Go to **Services** -> Select **Cloudformation**
5) Select **Create Stack** -> Select **With New resources**
6) Upload the template downloaded in step 1
7) Click Next and Provide a name for the stack (Ex : my-inferences). You can modify the Sagemaker Inference end-point name if required. (Leave it default for the lab)
8) Click Next and Launch the stack
9) After the Stack is completely created. Check the output of the stack to see the public endpoint created. Copy the endpoint to test

![Cloudformation Output](images/my-inferences-cfn-output.PNG)


## Step 2: Test the API Endpoint

In this step we will use a external Rest API tool like Rest Client or Postman to call the endpoint

1) Download and install the chrome pugin for "Advanced Rest Client"
2) Open the Advanced Rest Client
3) Provide the Endpoint URL from the previous step and select the method as Post
4) Provide the Post body as JSON **{"data": "5.0,3.5,1.3,0.3"}** (Example input format for Iris Classification)

![Rest Client Input](images/rest_client_input.PNG)

5) Verify the output prediction

![Rest Client Input](images/rest_client_output.PNG)

---

## Step 5: Clean-Up

If you are performing work in your own AWS Account, please clean up resources you will no longer use to avoid unnecessary charges by deleting the cloudformation stacks. 

