pipeline {

    agent any

    environment {
        AWS_ECR_LOGIN = 'true'
        END_POINT = 'jenkins-scikit-byo'
	SAGEMAKER_TRAINING_JOB = 'jenkins-scikit-byo'
	LAMBDA_EVALUATE_MODEL = 'MLOps-InvokeEndpoint-scikitbyo'
	TRAIN_FILE = 'train.csv'
	TEST_FILE = 'test.csv'
	AWS_REGION = 'us-east-1'
    }

    stages {
        stage("Checkout") {
            steps {
               checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[url: 'https://github.com/chethancmk/mlops-sagemaker-jenkins-byo']]])
            }
        }

        stage("BuildPushContainer") {
            steps {
              sh """
                echo "${params.ECRURI}"
                aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ${params.ECRURI}
 	              docker build -t scikit-byo:${env.BUILD_ID} .
                docker tag scikit-byo:${env.BUILD_ID} ${params.ECRURI}:${env.BUILD_ID} 
                docker push ${params.ECRURI}:${env.BUILD_ID}
              """
            }
        }
        
        stage("TrainModel") {
            steps { 
              sh """
               aws sagemaker create-training-job --training-job-name ${env.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID} \
	       --algorithm-specification TrainingImage="${params.ECRURI}:${env.BUILD_ID}",TrainingInputMode="File" \
	       --role-arn ${params.SAGEMAKER_EXECUTION_ROLE_TEST} \
	       --input-data-config '{"ChannelName": "training", "DataSource": { "S3DataSource": { "S3DataType": "S3Prefix", "S3Uri": "s3://${params.S3_DATA_BUCKET}/${env.TRAIN_FILE}"}}}' \
	       --resource-config InstanceType='ml.c4.2xlarge',InstanceCount=1,VolumeSizeInGB=5 \
	       --output-data-config S3OutputPath='${params.S3_MODEL_ARTIFACTS}' \
	       --stopping-condition MaxRuntimeInSeconds=3600 \
	       --region='${env.AWS_REGION}'
              """
             }
        }

      stage("TrainStatus") {
            steps {
              script {
                    def response = sh """ 
                    TrainingJobStatus=`aws sagemaker describe-training-job --training-job-name \"${env.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}\" | grep -Po \'"\'"TrainingJobStatus"\'"\\s*:\\s*"\\K([^"]*)\'`
                    echo \$TrainingJobStatus
                    while [ \$TrainingJobStatus = "InProgress" ] ; do
                      TrainingJobStatus=`aws sagemaker describe-training-job --training-job-name \"${env.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}\" | grep -Po \'"\'"TrainingJobStatus"\'"\\s*:\\s*"\\K([^"]*)\'`
                      echo \$TrainingJobStatus
                      sleep 1m
                    done
                    """
                    
                  }
              }
      }

      stage("DeployToTest") {
            steps { 
              sh """
               if ! aws cloudformation describe-stacks --region us-east-1 --stack-name '${env.SAGEMAKER_TRAINING_JOB}'-test ; then
                  echo -e "\nStack does not exist, creating ..."
                  aws cloudformation create-stack --region us-east-1 --stack-name '${env.SAGEMAKER_TRAINING_JOB}'-test --template-body file://deploy/cfn-sagemaker-endpoint.yml --parameters  ParameterKey=ModelName,ParameterValue=\"${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}\" ParameterKey=ModelDataUrl,ParameterValue=\"${S3_MODEL_ARTIFACTS}/${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}\"/output/model.tar.gz ParameterKey=TrainingImage,ParameterValue="${params.ECRURI}:${env.BUILD_ID}" ParameterKey=InstanceType,ParameterValue='ml.t2.large'  ParameterKey=InstanceCount,ParameterValue='1' ParameterKey=RoleArn,ParameterValue="${params.SAGEMAKER_EXECUTION_ROLE_TEST}" ParameterKey=Environment,ParameterValue='Test'
                  echo "Waiting for stack to be created ..."
                  aws cloudformation wait stack-create-complete --region us-east-1 --stack-name "${env.SAGEMAKER_TRAINING_JOB}"-test
               else
                  echo -e '\nStack exists, attempting update ...'
                  set +e
                  update_output=`aws cloudformation update-stack --region us-east-1 --stack-name '${env.SAGEMAKER_TRAINING_JOB}'-test --template-body file://deploy/cfn-sagemaker-endpoint.yml --parameters  ParameterKey=ModelName,ParameterValue=\"${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}\" ParameterKey=ModelDataUrl,ParameterValue=\"${S3_MODEL_ARTIFACTS}/${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}\"/output/model.tar.gz ParameterKey=TrainingImage,ParameterValue="${params.ECRURI}:${env.BUILD_ID}" ParameterKey=InstanceType,ParameterValue='ml.t2.large'  ParameterKey=InstanceCount,ParameterValue='1' ParameterKey=RoleArn,ParameterValue="${params.SAGEMAKER_EXECUTION_ROLE_TEST}" ParameterKey=Environment,ParameterValue='Test'`
                  status=\$?
                  set -e
                  echo \$update_output
                  if [ \$status -ne 0 ] ; then
                  # Don't fail for no-op update
                    if [[ \$update_output == *"ValidationError"* && \$update_output == *"No updates"* ]] ; then
                      echo -e "\nFinished create/update - no updates to be performed"
                      exit 0
                    else
                      exit \$status
                    fi
                  fi

               echo "Waiting for stack update to complete ..."
               aws cloudformation wait stack-update-complete --region us-east-1 --stack-name '${env.SAGEMAKER_TRAINING_JOB}'-test

               fi
               echo "Finished create/update successfully!"
              """
             }
        }
	    
	stage("TestEvaluate") {
		    steps { 
			script {
					    sh 'echo "Invoking Lambda for Testing Endpoint"'
						result = invokeLambda(
								functionName: "${env.LAMBDA_EVALUATE_MODEL}" ,
								payload: [ "EndpointName": "${env.END_POINT}-Test","Env": "Test", "S3TestData":  "${params.S3_DATA_BUCKET}", "S3Key": "${env.TEST_FILE}" ],
								returnValueAsString: true
						)
						if (result.contains("success")){
						   echo 'The Test Endpoint has Succeded'					    
						}
						else{
						    error 'The Test End Point did not succeed'
						}
					}
		    }
		      }
	

      stage("DeployToProd") {
            steps { 
              sh """
               if ! aws cloudformation describe-stacks --region us-east-1 --stack-name '${env.SAGEMAKER_TRAINING_JOB}'-prod ; then
                  echo -e "\nStack does not exist, creating ..."
                  aws cloudformation create-stack --region us-east-1 --stack-name '${env.SAGEMAKER_TRAINING_JOB}'-prod --template-body file://deploy/cfn-sagemaker-endpoint.yml --parameters  ParameterKey=ModelName,ParameterValue=\"${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}\" ParameterKey=ModelDataUrl,ParameterValue=\"${S3_MODEL_ARTIFACTS}/${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}\"/output/model.tar.gz ParameterKey=TrainingImage,ParameterValue="${params.ECRURI}:${env.BUILD_ID}" ParameterKey=InstanceType,ParameterValue='ml.t2.large'  ParameterKey=InstanceCount,ParameterValue='1' ParameterKey=RoleArn,ParameterValue="${params.SAGEMAKER_EXECUTION_ROLE_TEST}" ParameterKey=Environment,ParameterValue='Prod'
                  echo "Waiting for stack to be created ..."
                  aws cloudformation wait stack-create-complete --region us-east-1 --stack-name "${env.SAGEMAKER_TRAINING_JOB}"-prod
               else
                  echo -e '\nStack exists, attempting update ...'
                  set +e
                  update_output=`aws cloudformation update-stack --region us-east-1 --stack-name '${env.SAGEMAKER_TRAINING_JOB}'-prod --template-body file://deploy/cfn-sagemaker-endpoint.yml --parameters  ParameterKey=ModelName,ParameterValue=\"${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}\" ParameterKey=ModelDataUrl,ParameterValue=\"${S3_MODEL_ARTIFACTS}/${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}\"/output/model.tar.gz ParameterKey=TrainingImage,ParameterValue="${params.ECRURI}:${env.BUILD_ID}" ParameterKey=InstanceType,ParameterValue='ml.t2.large'  ParameterKey=InstanceCount,ParameterValue='1' ParameterKey=RoleArn,ParameterValue="${params.SAGEMAKER_EXECUTION_ROLE_TEST}" ParameterKey=Environment,ParameterValue='Prod'`
                  status=\$?
                  set -e
                  echo \$update_output
                  if [ \$status -ne 0 ] ; then
                  # Don't fail for no-op update
                    if [[ \$update_output == *"ValidationError"* && \$update_output == *"No updates"* ]] ; then
                      echo -e "\nFinished create/update - no updates to be performed"
                      exit 0
                    else
                      exit \$status
                    fi
                  fi
               echo "Waiting for stack update to complete ..."
               aws cloudformation wait stack-update-complete --region us-east-1 --stack-name '${env.SAGEMAKER_TRAINING_JOB}'-prod
               fi
               echo "Finished create/update successfully!"
              """
             }
        }

  }
}   
