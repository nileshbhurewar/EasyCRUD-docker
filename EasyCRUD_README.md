
# EasyCRUD Dockerized Project

This project demonstrates a complete CI/CD pipeline using **Jenkins**, **Docker**, **MariaDB**, and **AWS EC2** to build, push, and deploy frontend and backend applications.

---

## 🚀 Project Overview

- Frontend and Backend built with Docker  
- Database: MariaDB  
- CI/CD Pipeline managed by Jenkins  
- Images pushed to Docker Hub  
- Deployment on AWS EC2 Instance  

---

## ✅ Prerequisites

- AWS EC2 instance (example IP: `35.154.244.253`)  
- Open port `3306` for MariaDB in EC2 Security Group  
- Docker Hub account  
- Jenkins installed on EC2  

---

## ⚡ Installation Steps

### 1️⃣ Install Java
```bash
sudo apt update
sudo apt install fontconfig openjdk-21-jre
java -version
```

### 2️⃣ Install Jenkins
```bash
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins
```

### 3️⃣ Install Docker
```bash
sudo apt install docker.io -y
sudo usermod -aG docker jenkins
```

### 4️⃣ Grant Jenkins Sudo Privileges
```bash
sudo visudo
# Add the following line at the end of the file:
jenkins ALL=(ALL) NOPASSWD: ALL
```

### 5️⃣ Install MariaDB
```bash
sudo apt update
sudo apt install mariadb-server -y
sudo mysql_secure_installation
```

---

## ✅ Create Database and User

```sql
CREATE DATABASE student_db;
GRANT ALL PRIVILEGES ON student_db.* TO 'root'@'%' IDENTIFIED BY 'redhat';
FLUSH PRIVILEGES;
```

---

## ✅ Application Configuration

### `application.properties`
```properties
server.port=8081
spring.datasource.url=jdbc:mysql://35.154.244.253:3306/student_db
spring.datasource.username=root
spring.datasource.password=redhat
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MariaDBDialect
```

### `.env`
```bash
VITE_API_URL="http://35.154.244.253:8081/api"
```

---

## ✅ Docker Cleanup Command

```bash
docker kill $(docker ps -q) && docker rm -v $(docker ps -a -q) && docker rmi $(docker images -q)
```

---

## ✅ Store DockerHub Credentials in Jenkins

1. Go to Jenkins Dashboard → Manage Jenkins → Manage Credentials  
2. Select domain: `(global)`  
3. Add Credentials:  
    - **Username:** `<dockerhub-username>`  
    - **Password:** `<dockerhub-password>`  
    - **ID:** `dockerhub-cred`  

---

## ✅ Jenkins Pipeline Overview

```groovy
pipeline {
    agent any

    stages {
        stage('Clone Repo') {
            steps {
                git url: "https://github.com/nileshbhurewar/EasyCRUD-docker", branch: "main"
            }
        }

        stage('Build Frontend Image') {
            steps {
                sh "docker build -t nileshbhurewar/easycrud1-jenkins:frontend ./frontend"
            }
        }

        stage('Build Backend Image') {
            steps {
                sh "docker build -t nileshbhurewar/easycrud1-jenkins:backend ./backend"
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
                sh "docker push nileshbhurewar/easycrud1-jenkins:frontend"
            }
        }

        stage('Push Backend Image to Docker Hub') {
            steps {
                sh "docker push nileshbhurewar/easycrud1-jenkins:backend"
            }
        }

        stage('Run Frontend Container') {
            steps {
                sh '''
                    docker rm -f easycrud1-frontend || true
                    docker run -d --name easycrud1-frontend -p 80:80 nileshbhurewar/easycrud1-jenkins:frontend
                '''
            }
        }

        stage('Run Backend Container') {
            steps {
                sh '''
                    docker rm -f easycrud1-backend || true
                    docker run -d --name easycrud1-backend -p 8081:8081 nileshbhurewar/easycrud1-jenkins:backend
                '''
            }
        }
    }
}
```

---

## ✅ Notes

- Ensure EC2 Security Group allows these ports:  
    - `3306` (MariaDB)  
    - `80` (Frontend)  
    - `8081` (Backend)  

- Store DockerHub credentials in Jenkins for secure pipeline execution.

---

## 📞 Contact

For any issues, raise an issue in the repository or contact me.

---

🌟 Happy Coding!
