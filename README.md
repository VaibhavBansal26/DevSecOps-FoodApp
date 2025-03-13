# DevSecOps-FoodApp
Deploying Food Delivery App - Terraform, Jenkins, SonarQube, Trivy, Docker, Prometheus, Grafana, Helm, React Js

System Architecture

![App](https://res.cloudinary.com/vaibhav-codexpress/image/upload/v1741897056/diagram-export-13-03-2025-16_15_50_nrr0sx.png)


App: http://35.173.218.166:3000/

Jenkins: http://35.173.218.166:8080/

SonarQube: http://35.173.218.166:9000/


# Flow

# Jenkins

![jenkins](https://res.cloudinary.com/vaibhav-codexpress/image/upload/v1741897012/Screenshot_2025-03-13_at_3.52.24_PM_ejwjun.png)

# SonarQube

![sonarqube](https://res.cloudinary.com/vaibhav-codexpress/image/upload/v1741897013/Screenshot_2025-03-13_at_3.59.07_PM_tgfg9h.png)

# App

![app](https://res.cloudinary.com/vaibhav-codexpress/image/upload/v1741897011/Screenshot_2025-03-13_at_3.52.36_PM_b1lp4q.png)




Creation of IAM User
Terraform SCript -> aws configure
terraform init

```
brew install terraform
which terraform
source ~/.zshrc

```

```
terraform plan # information of the resources we are going to create
```

```
terraform apply --auto-approve
```

root user

```
sudo su

 whoami
```

Verify Jenkins is running

```
sudo systemctl status jenkins
```

Configure Jenkins Installations (Plugins)

Configure Jeninkins Tools
1. jdk17
2. SonarQubeScanner
3. Nodejs
4. Owasp DP-cHECK
5. GIT
6. Docker
   
Configure Sonarqube - 

Create Token - Administration -> Security -> Users -> Token -> Generate Token

Configure Tokens in Jenkins Dashboard to connect everything

Add Tokens to Jenkins -> Managae Jenkins -> Credentials

Add Sonar Token,Docker Cred

Create Sonarqube webhook

Configure sonarqube url in jenkins

Configure System - Configure Global Settings

1. Configure sonarqube url (sonarqube server)


Jenkins Pipeline Script

```
pipeline{
    agent any
    tools{
        jdk 'jdk17'
        nodejs 'node23'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout from Git'){
            steps{
                git 'https://github.com/VaibhavBansal26/DevSecOps-FoodApp.git'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=FoodApp \
                    -Dsonar.projectKey=FoodApp '''
                }
            }
        }
        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' 
                }
            } 
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker'){   
                       sh "docker build -t swiggy ."
                       sh "docker tag swiggy vaibhavbansal26/swiggy:latest "
                       sh "docker push vaibhavbansal26/swiggy:latest "
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image vaibhavbansal26/swiggy:latest > trivy.txt" 
            }
        }
        stage('Deploy to container'){
            steps{
                sh 'docker run -d --name swiggy -p 3000:3000 vaibhavbansal26/swiggy:latest'
            }
        }
    }
}

```