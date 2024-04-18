==============================================
     DOCKER_ECS_ECR CICD Project Overview
==============================================

Requirements:
-------------
1. IAM user and role with appropriate ECR and ECS permissions.
2. Store AWS credentials securely in Jenkins.
3. Setup ECR and ECS on AWS.
4. Install necessary plugins like Docker pipeline on Jenkins.
5. Install Docker and AWS CLI on the Jenkins server.
6. Install additional plugins for ECR and ECS management.

Steps:
------
1. Install Docker Engine on Jenkins Server:
   - Add the Jenkins user to the docker group.
   - Reboot the server to apply changes.

2. Install AWS CLI:
   - Ensure AWS CLI is installed for managing AWS services via command line.

3. Create IAM User:
   - Setup an IAM user with necessary permissions for ECR and ECS access.

4. Create ECR Repository:
   - Provision an ECR repository to store Docker images.

5. Create ECS:
   - Configure ECS services and task definitions for deploying containers.

6. Install Jenkins Plugins:
   - ECR: For managing Docker images in AWS ECR.
   - Docker Pipeline: For defining and using Docker containers in Jenkins pipelines.
   - AWS SDK for managing credentials and other AWS services.

7. Store AWS Credentials in Jenkins:
   - Securely store and manage AWS credentials using Jenkins credential management.

8. Run the Pipeline:
   - Execute the Jenkins pipeline to build, push Docker images to ECR, and deploy to ECS.

==============================================
Ensure security best practices by limiting IAM permissions to what is necessary for operations. Regularly update Jenkins and its plugins to the latest versions to avoid security vulnerabilities.
==============================================



pipeline {
    agent any
    tools {
	    maven "MAVEN3"
	    jdk "OracleJDK11"
	}

    environment {
        registryCredential = 'ecr:us-east-1:awscreds'
        appRegistry = "590183974589.dkr.ecr.us-east-1.amazonaws.com/vprofileappimg"
        vprofileRegistry = "https://590183974589.dkr.ecr.us-east-1.amazonaws.com"
    }
  stages {
    stage('Fetch code'){
      steps {
        git branch: 'docker', url: 'https://github.com/devopshydclub/vprofile-project.git'
      }
    }


    stage('Test'){
      steps {
        sh 'mvn test'
      }
    }

    stage ('CODE ANALYSIS WITH CHECKSTYLE'){
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }

        stage('build && SonarQube analysis') {
            environment {
             scannerHome = tool 'sonar4.7'
          }
            steps {
                withSonarQubeEnv('sonar') {
                 sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                   -Dsonar.projectName=vprofile-repo \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }
            }
        }


    stage('Build App Image') {
       steps {
       
         script {
                dockerImage = docker.build( appRegistry + ":$BUILD_NUMBER", "./Docker-files/app/multistage/")
             }

     }
    
    }

    stage('Upload App Image') {
          steps{
            script {
              docker.withRegistry( vprofileRegistry, registryCredential ) {
                dockerImage.push("$BUILD_NUMBER")
                dockerImage.push('latest')
              }
            }
          }
     }

  }
}
