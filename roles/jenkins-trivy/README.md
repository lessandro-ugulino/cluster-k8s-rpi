# Services: Jenkins & Trivy

## Index

- [Summary](#summary)
- [Requirements](#requirements)
- [Setup](#how-to-use)
  - [Jenkins](#jenkins)
  - [Trivy](#trivy)
    - [Build an image and scan it using trivy](#build-an-image-and-scan-it-using-trivy)
- [Diagram](#diagram)

## Summary

This pipeline will deploy Jenkins and Trivy.

**Jenkins**
_Jenkins is an open source automation server. It helps automate the parts of software development related to building, testing, and deploying, facilitating continuous integration and continuous delivery._

**Trivy**
_Trivy is a comprehensive and easy-to-use open source vulnerability scanner for container images_

## Requirements

1. Before deploy this application, make sure you've deployed the **role/k8s/service-acc**, `jenkins-admin`.

## Setup

### Jenkins

Access http://192.168.1.111:32000, and grep the password requested to unlock Jenkins using the command below. Make sure you use your pod name.

```bash
kubectl exec -ti jenkins-7d64cb8665-2mj2g -n devops-tools -- cat /var/jenkins_home/secrets/initialAdminPassword
```

![jenkins-unlock](../../img/jenkins-unlock.png)

Click **Install suggested plugins**

![jenkins-customize](../../img/jenkins-customize.png)

Wait for the installation to complete.

![jenkins-plugin](../../img/jenkins-plugin.png)

Setup the admin user and password.

![jenkins-setup](../../img/jenkins-setup.png)

### Trivy

Basically, the Jenkins pipeline will install trivy on the Jenkins container if it's not installed.

<a href= https://github.com/aquasecurity/trivy>Documentation </a>

### Build an image and scan it using trivy

**Required Jenkins plugins**

1. <a href=https://plugins.jenkins.io/docker-workflow> Docker Pipeline</a>

2. <a href=https://plugins.jenkins.io/aws-java-sdk-ecr> AWS ECR</a>

**Jenkinsfile and Dockerfile**

Create a _Jenkinsfile_ and a _Dockerfile_ similar on this repo.

Basically, this Jenkins pipeline will build an image using the _Dockerfile_, install **trivy** on Jenkins container if it's not already installed, scan it via **trivy** to find any vulnerability and upload it to ECR Repo if no vulnerability was found.

Dockerfile

```
FROM alpine:latest
LABEL maintainer="lezampieri@hotmail.com"
```

Jenkinsfile

```
pipeline {
    environment {
        ECR_TAG_PREFIX = "111111111111.dkr.ecr.ap-southeast-2.amazonaws.com/demo"
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
                docker.withRegistry(ECR, 'ecr:ap-southeast-2:aws_credentials') {
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
```

On Jenkins console create a _Multibranch Pipeline_

![jenkins-project](../../img/jenkins-project.png)

Make sure you added the Repo URL address.

![jenkins-github](../../img/jenkins-github.png)

Click on **Build with Parameters**

![jenkins-build](../../img/jenkins-build.png)

Select the ECR repo to upload the image and click **Build**

![jenkins-repo](../../img/jenkins-repo.png)

You can check the output log by clicking on **Console Output**

![jenkins-logs](../../img/jenkins-logs.png)

If trivy scan finds any **HIGH** vulnerability, it'll stop the pipeline as shown below.

![jenkins-trivy-error](../../img/jenkins-trivy-error.png)

Otherwise, it'll upload the image created from Dockerfile.

![jenkins-trivy](../../img/jenkins-trivy.png)

![jenkins-done](../../img/jenkins-done.png)

## Diagram

![jenkins-diagram](../../img/jenkins-diagram.png)
