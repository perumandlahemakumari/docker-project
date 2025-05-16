
# 🚀 Docker End-to-End DevSecOps Project

## 📋 Step-by-Step Instructions

### 🖥️ Step 1: Launch EC2 Instance
- 💻 Instance Type: `t2.large`
- 💾 Volume Size: `25GB`

### ⚙️ Step 2: Install Jenkins, Git, Docker, and Trivy

#### 🔍 Trivy Installation:
```bash
wget https://github.com/aquasecurity/trivy/releases/download/v0.18.3/trivy_0.18.3_Linux-64bit.tar.gz
tar xvf trivy_0.18.3_Linux-64bit.tar.gz
sudo mv trivy /usr/local/bin/
export PATH=$PATH:/usr/local/bin/
source ~/.bashrc
trivy --version
```

#### 🛠️ Jenkins Installation:
```bash
yum install java-17-amazon-corretto -y
sudo wget -O /etc/yum.repos.d/jenkins.repo \
    https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
yum install jenkins -y
systemctl start jenkins
```

#### 🔧 Git & Docker Installation:
```bash
yum install git docker -y
systemctl start docker
chmod 777 /var/run/docker.sock
```

---

### 📦 Step 3: Install Jenkins Plugins
- 📌 Sonar Scanner
- 📌 NodeJs
- 📌 OWASP Dependency Check
- 📌 Docker Pipeline
- 📌 Eclipse Temurin Installer Version
- 📌 Pipeline Stage View

---

### 🔗 Step 4: Plugin Configuration in Jenkins

#### 🐳 Setup SonarQube Using Docker:
```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```

🔑 Access Sonar Dashboard:
- 🌐 URL: `http://<EC2-IP>:9000`
- 👤 Username: `admin`
- 🔐 Password: `admin`

🛠️ Create a dummy project and generate a security token.

#### 🧩 Configure Plugins in Jenkins:
- ➕ Add credentials → Secret Text → ID: `sonar`
- ⚙️ Manage Jenkins → Configure System:
  - Add Sonar Server → Name: `mysonar`
  - Enable auto-install

🛠️ Add Tools:
- ☕ JDK → Name: `jdk17` → Version: `jdk-17.0.8.1+1`
- 🟢 NodeJS → Name: `node16` → Version: `NodeJS 16.2.0`
- 🛡️ Dependency Check → Name: `DP-Check` → Version: `6.5.1`

---

## 🧪 Jenkins Pipeline Script

```groovy
pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }

    environment {
        SCANNER_HOME = tool 'mysonar'
    }

    stages {

        stage('🧹 Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('📥 Checkout Code') {
            steps {
                git 'https://github.com/suneelprojects/docker-project.git'
            }
        }

        stage('🔍 SonarQube Analysis') {
            steps {
                withSonarQubeEnv('mysonar') {
                    sh '''
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectName=zomato \
                        -Dsonar.projectKey=zomato \
                        -Dsonar.sources=.
                    '''
                }
            }
        }

        stage('📦 Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('🛡️ OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('🧪 Trivy FS Scan') {
            steps {
                sh 'trivy fs . > trivyfs.txt'
            }
        }

        stage('🏗️ Build Docker Image') {
            steps {
                sh 'docker build -t zomato .'
            }
        }

        stage('🏷️ Tag & 🚀 Push Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker') {
                        sh 'docker tag zomato hemakumariperumandla/docker-project-may:zomatov1'
                        sh 'docker push hemakumariperumandla/docker-project-may:zomatov1'
                    }
                }
            }
        }

        stage('🔍 Trivy Image Scan') {
            steps {
                sh 'trivy image hemakumariperumandla/docker-project-may:zomatov1'
            }
        }

        stage('▶️ Run Docker Container') {
            steps {
                sh 'docker run -itd --name zomato -p 3000:3000 hemakumariperumandla/docker-project-may:zomatov1'
            }
        }
    }
}
```
