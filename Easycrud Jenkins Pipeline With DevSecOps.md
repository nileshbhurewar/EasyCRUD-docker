# EasyCRUD Jenkins Pipeline Deployment with DevSecOps

This project demonstrates a **complete CI/CD pipeline** for a Spring Boot + React EasyCRUD application using **Jenkins, Docker, SonarQube, OWASP Dependency Check, Trivy, AWS S3, and Docker Hub**. The pipeline automates code analysis, artifact packaging, S3 upload, Docker image build & deployment, and email notifications.

---

## **1. EC2 Setup (t2.large)**

1. **Install Java 17**

```bash
sudo apt install openjdk-17-jdk -y
java -version
```

2. **Install Docker**

```bash
sudo apt update -y
sudo apt install docker.io -y
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker ubuntu
newgrp docker
```

3. **Install Maven**

```bash
sudo apt install maven -y
mvn -version
```

4. **Install AWS CLI**

```bash
sudo snap install aws-cli --classic
aws --version
aws configure
```

5. **Install MySQL client**

```bash
sudo apt install mysql-client -y
```

6. **Install Trivy**

```bash
sudo apt-get install -y wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install -y trivy
trivy --version
```

---

## **2. SonarQube Setup (Docker)**

```bash
docker run -d --name sonarqube-custom -p 9000:9000 sonarqube:10.6-community
```

* Access SonarQube on `http://<EC2_PUBLIC_IP>:9000`
* **Create Webhook for Jenkins**

  * Go to `Administration → Configuration → Webhooks`
  * URL: `http://<JENKINS_URL>:8080/sonarqube-webhook/`
* **Generate Admin Token**

  * `Administration → Security → Users → Token → Generate`

---

## **3. AWS RDS Setup**

* **DB Name:** `student_db`
* **Username:** `admin`
* **Password:** `Redhat123`
* **Endpoint:** `database-1.cxi2eu6c2mr6.ap-south-1.rds.amazonaws.com`

---

## **4. Jenkins Setup**

1. **Install Required Plugins**

   * OWASP Dependency Check
   * Docker Pipeline
   * SonarQube Scanner
   * SonarQube Quality Gates
   * Pipeline Stage View
   * S3 Publisher

2. **Configure SonarQube in Jenkins**

   * `Manage Jenkins → System → SonarQube Servers`
   * Add **SonarQube token** for authentication

3. **Configure Quality Gates**

   * `Manage Jenkins → Tools → SonarQube Scanner installations`

4. **Configure OWASP Dependency Check**

   * `Jenkins → Tools → Dependency-Check installations`
   * Install automatically from GitHub

5. **Configure Trivy** (already installed on EC2)

6. **AWS S3 Credentials**

   * `Manage Jenkins → Credentials → Global → Add Credentials`
   * ID: `aws-creds`
   * Bucket ARN: `arn:aws:s3:::easycrud-artifacts`

7. **Docker Hub Credentials**

   * ID: `dockerhub-cred`

8. **Email Notification**

   * `Manage Jenkins → Configure System → Extended E-mail Notification`
   * SMTP Server: `smtp.gmail.com`
   * Port: 465
   * Credentials: Gmail username & app password

---

## **5. Jenkins Pipeline (Declarative)**

```groovy
pipeline {
    agent any

    environment {
        SONAR_HOME = tool "sonar"                  
        S3_BUCKET = "easycrud-artifacts"          
        FRONTEND_IMAGE = "nileshbhurewar/easycrud1-jenkins:frontend"  
        BACKEND_IMAGE  = "nileshbhurewar/easycrud1-jenkins:backend"   
        DOCKER_HUB_CREDENTIALS = "dockerhub-cred"
    }

    stages {
        stage('Clone Repo') {
            steps {
                git url: "https://github.com/nileshbhurewar/EasyCRUD-docker.git", branch: "main"
            }
        }

        stage('SonarQube Code Analysis') {
            steps {
                dir('backend') {
                    withSonarQubeEnv('sonar') {
                        sh '''
                            mvn clean verify sonar:sonar \
                              -Dsonar.projectKey=easycrud \
                              -Dsonar.projectName=easycrud \
                              -Dsonar.host.url=$SONAR_HOST_URL
                        '''
                    }
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--scan ./', odcInstallation: 'dc'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('SonarQube Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build & Package Artifacts') {
            steps {
                dir('backend') {
                    sh '''
                        mvn clean package -DskipTests
                        mkdir -p artifacts
                        cp target/*.jar artifacts/
                    '''
                }
            }
        }

        stage('Upload Artifacts to S3') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'aws-creds',
                                                  usernameVariable: 'AWS_ACCESS_KEY_ID',
                                                  passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh '''
                        aws s3 cp backend/artifacts/ s3://easycrud-artifacts/ \
                        --recursive --exclude "*" --include "*.jar" \
                        --region ap-south-1
                    '''
                }
            }
        }

        stage('Build Frontend Image') {
            steps {
                sh "docker build -t ${FRONTEND_IMAGE} ./frontend"
            }
        }

        stage('Build Backend Image') {
            steps {
                sh "docker build -t ${BACKEND_IMAGE} ./backend"
            }
        }

        stage('Docker Hub Login') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-cred', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                    sh "echo $PASSWORD | docker login -u $USERNAME --password-stdin"
                }
            }
        }

        stage('Push Frontend Image to Docker Hub') {
            steps {
                sh "docker push ${FRONTEND_IMAGE}"
            }
        }

        stage('Push Backend Image to Docker Hub') {
            steps {
                sh "docker push ${BACKEND_IMAGE}"
            }
        }

        stage('Run Frontend Container') {
            steps {
                sh '''
                    docker rm -f easycrud1-frontend || true
                    docker run -d --name easycrud1-frontend -p 80:80 ${FRONTEND_IMAGE}
                '''
            }
        }

        stage('Run Backend Container') {
            steps {
                sh '''
                    docker rm -f easycrud1-backend || true
                    docker run -d --name easycrud1-backend -p 8081:8081 ${BACKEND_IMAGE}
                '''
            }
        }
    }

    post {
        success {
            emailext(
                subject: " Build Success: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                    Build Successful! 🎉<br>
                    Job: ${env.JOB_NAME}<br>
                    Build Number: ${env.BUILD_NUMBER}<br>
                    Docker images built, pushed, and containers deployed.<br>
                    Artifacts uploaded to S3.
                """,
                to: "nileshbhurewar@gmail.com",
                mimeType: 'text/html'
            )
        }
        failure {
            emailext(
                subject: " Build Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                    Build Failed! <br>
                    Job: ${env.JOB_NAME}<br>
                    Build Number: ${env.BUILD_NUMBER}<br>
                    Check Jenkins console for details.
                """,
                to: "nileshbhurewar@gmail.com",
                mimeType: 'text/html'
            )
        }
        always {
            archiveArtifacts artifacts: '**/*.html, **/*.xml', fingerprint: true
        }
    }
}

```
