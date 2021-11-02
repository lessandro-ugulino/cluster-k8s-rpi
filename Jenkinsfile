pipeline {
    environment {
        ECR_TAG_PREFIX = "111111111111.dkr.ecr.ap-southeast-2.amazonaws.com"
        ECR = "https://${ECR_TAG_PREFIX}"
  }
  agent any
  parameters {
    choice choices: ['demo','repo1','repo2'],
      description: 'ECR repo',
      name: 'ECR_REPO'
  }
  stages{
    stage('Building image') {
        steps{
            script {
                dockerImage = docker.build "${params.ECR_REPO}:${BUILD_NUMBER}"
            }
        }
    }
    stage('Install Trivy') {
        steps {
            sh(returnStdout: true, script: '''#!/bin/bash
            dpkg -s trivy &> /dev/null
            if [ $? -ne 0 ]
                then
                    echo "trivy is not installed"
                    curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin v0.20.2
                else
                    echo "trivy is installed"
            fi
            '''.stripIndent())
        }
    }
    stage("Trivy Scan") {
        steps {
            script {
                sh '''#!/bin/bash
                echo "Trivy image scanning started...."

                echo "Scanning MEDIUM severity..."
                trivy --severity MEDIUM "${ECR_REPO}:${BUILD_NUMBER}"

                echo "Scanning HIGH and CRITICAL severity..."
                trivy --exit-code 1 --severity HIGH,CRITICAL "${ECR_REPO}:${BUILD_NUMBER}"

                # Trivy scan result processing
                my_exit_code=$?
                echo "RESULT 1:---> $my_exit_code"

                # Check scan results
                if [[ $my_exit_code -eq 1 ]]
                then
                    echo "Image scanning failed. Some vulnerabilities found"
                    exit 1;
                else
                    echo "Image is scanned Successfully. No vulnerabilities found"
                fi
                '''
            }
        }
    }
    stage('Docker push') {
        steps {
            script {
                docker.withRegistry(ECR/ECR_REPO, 'ecr:ap-southeast-2:aws_credentials') {
                dockerImage.push()
                dockerImage.push 'latest'
                }
            }
        }
    }
  }
  post {
    always {
      deleteDir()
    }
  }
}