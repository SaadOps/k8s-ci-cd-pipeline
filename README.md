# JENKINS-END-TO-END-CICD-Implementation
![2024-04-16 11 56 42 web telegram org 2468f1127a03](https://github.com/SaadOps/JENKINS-END-TO-END-CICD-Implementation/assets/94478736/80947195-4371-40c0-97b7-82dc19a92ea1)

# Table of Contents
- [Continuous Integration](#continuous-integration)
- [Continuous Delivery (CI)](#continuous-delivery-ci)

## Continuous Integration
1. Step up t2.medium EC2 Instance from AWS Management Console.
2. SSH to the EC2
    ```bash
    sudo apt update
    sudo apt install openjdk-11-jre
    apt install unzip
    ```
3. Set up Jenkins on the EC2 Instance.
    ```bash
    curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
    echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
    sudo apt-get update 
    sudo apt-get install jenkins
    ```
4. Configure the Security Group Attached to the EC2 Instance and Allow inbound Traffic to TCP Port: 8080 (Default Port for Jenkins) and 9000 (Default Port for Sonarqube).
5. Access Jenkins using: `<EC2 IP ADDRESS>:8000/`
6. The default ID for Jenkins is `admin` and the Process to get the Password is:
    ```bash
    sudo cat /var/lib/jenkins/secrets/initialAdminPassword
    ```
7. Configure Jenkins:
    - Install Suggested Plugins.
    - Install Docker Pipeline Plugin.

8. Docker Slave Configuration on EC2 Instance.
    ```bash
    sudo apt install docker.io
    sudo su -
    usermod -aG docker jenkins
    usermod -aG docker ubuntu
    sudo chmod 666 /var/run/docker.sock
    systemctl restart docker
    ```

9. Install and Configure the Sonarqube Server on the EC2 Instance.
    ```bash
    adduser sonarqube
    sudo su - sonarqube
    wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.4.0.54424.zip
    unzip *
    chmod -R 755 /home/sonarqube/sonarqube-9.4.0.54424
    chown -R sonarqube:sonarqube /home/sonarqube/sonarqube-9.4.0.54424
    cd sonarqube-9.4.0.54424/bin/linux-x86-64/
    ./sonar.sh start
    ```

10. Generate the token from the SonarQube and provide it to Jenkins as a secret.

11. Write Jenkins Pipeline using JenkinsFile.
   ```groovy
pipeline {
  agent {
    docker {
      image 'saadf5/maven-saad-docker-agent:v1'
      args '--user root -v /var/run/docker.sock:/var/run/docker.sock' // mount Docker socket to access the host's Docker daemon
    }
  }
  stages {
    stage('Checkout') {
      steps {
        sh 'echo passed'
      }
    }
    stage('Build and Test') {
      steps {
        sh 'ls -ltr'
        // build the project and create a JAR file
        sh 'cd CICD/java-maven-sonar-argocd-helm-k8s/spring-boot-app && mvn clean package'
      }
    }
    stage('Static Code Analysis') {
      environment {
        SONAR_URL = "http://184.73.15.158:9000/"
      }
      steps {
        withCredentials([string(credentialsId: 'sonarkube', variable: 'SONAR_AUTH_TOKEN')]) {
          sh 'cd CICD/java-maven-sonar-argocd-helm-k8s/spring-boot-app && mvn sonar:sonar -Dsonar.login=$SONAR_AUTH_TOKEN -Dsonar.host.url=${SONAR_URL}'
        }
      }
    }
    stage('Build and Push Docker Image') {
      environment {
        DOCKER_IMAGE = "srasati19/ultimate-cicd:${BUILD_NUMBER}"
        // DOCKERFILE_LOCATION = "java-maven-sonar-argocd-helm-k8s/spring-boot-app/Dockerfile"
        REGISTRY_CREDENTIALS = credentials('docker-cred')
      }
      steps {
        script {
          sh 'cd CICD/java-maven-sonar-argocd-helm-k8s/spring-boot-app && docker build -t ${DOCKER_IMAGE} .'
          def dockerImage = docker.image("${DOCKER_IMAGE}")
          docker.withRegistry('https://index.docker.io/v1/', "docker-cred") {
            dockerImage.push()
          }
        }
      }
    }
    stage('Update Deployment File') {
      environment {
        GIT_REPO_NAME = "DevOps_"
        GIT_USER_NAME = "SaadOps"
      }
      steps {
        withCredentials([string(credentialsId: 'github', variable: 'GITHUB_TOKEN')]) {
          sh '''
            git config user.email "saad7teen@gmail.com"
            git config user.name "SaadOps"
            BUILD_NUMBER=${BUILD_NUMBER}
            sed -i "s/replaceImageTag/${BUILD_NUMBER}/g" CICD/java-maven-sonar-argocd-helm-k8s/spring-boot-app-manifests/deployment.yml
            git add CICD/java-maven-sonar-argocd-helm-k8s/spring-boot-app-manifests/deployment.yml
            git commit -m "Update deployment image to version ${BUILD_NUMBER}"
            git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME} HEAD:main
          '''
        }
      }
    }
  }
```
12. Create a new Pipeline, Define the Pipeline script from Source Code Management (SCM).
    - SCM: git
    - Path: Provide the Path location of the JenkinsFile

13. Run the Pipeline.

14. Sonarqube Dashboard.

## Continuous Delivery (CI)
1. Install Minikube on the local Machine: [Minikube Installation Guide](https://minikube.sigs.k8s.io/docs/start/)
    ```bash
    minikube start
    ```

2. Install and Configure Argo CD on the K8 Cluster.
    - [Argo CD Operator Link](https://operatorhub.io/operator/argocd-operator)
    - Perform all the steps in your K8 Cluster.
    - Create an Agro CD controller.
    - Change the service type of the argocd-server from Cluster IP to NodePort Mode.

3. Access the ArgoCD server using localhost:9000.
    - Default username: admin
    - Password: it is sorted in the secrets.

4. Create New App.
    - Provide the Git Repository URL, Manifest Path, Cluster URL, Namespace.

So we have Successfully deployed the Spring Boot-based application using the Argo CD operator and Kubernetes.

Argo-CD will continuously look for the Manifest file in the Repository, as soon as it finds the new Manifest file it will create a new deployment.

Here we have successfully implemented CICD Pipeline using various DevOps tools like Jenkins, ArgoCD, K8s, Docker, and AWS.


