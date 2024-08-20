# Deploy Netflix Clone on Cloud using Jenkins - DevSecOps Project

## Table of Contents
1. [Phase 1: Initial Setup and Deployment](#phase-1-initial-setup-and-deployment)
2. [Phase 2: Security](#phase-2-security)
3. [Phase 3: CI/CD Setup](#phase-3-cicd-setup)
4. [Phase 4: Monitoring](#phase-4-monitoring)
5. [Phase 5: Notification](#phase-5-notification)
6. [Phase 6: Kubernetes](#phase-6-kubernetes)

## Phase 1: Initial Setup and Deployment

### Step 1: Launch EC2 (Ubuntu 22.04)
- Provision an EC2 instance on AWS with Ubuntu 22.04.
- Connect to the instance using SSH.

### Step 2: Clone the Code
Update all packages and clone the code repository:

```bash
git clone https://github.com/N4si/DevSecOps-Project.git
```

### Step 3: Install Docker and Run the App Using a Container
Set up Docker on the EC2 instance:

```bash
sudo apt-get update
sudo apt-get install docker.io -y
sudo usermod -aG docker $USER  # Replace with your system's username, e.g., 'ubuntu'
newgrp docker
sudo chmod 777 /var/run/docker.sock
```

Build and run your application using Docker:

```bash
docker build -t netflix .
docker run -d --name netflix -p 8081:80 netflix:latest
```

To delete:

```bash
docker stop <containerid>
docker rmi -f netflix
```

**Note**: It will show an error because you need an API key.

### Step 4: Get the API Key
1. Navigate to TMDB (The Movie Database) website.
2. Create an account and log in.
3. Go to your profile and select "Settings."
4. Click on "API" from the left-side panel.
5. Create a new API key by clicking "Create" and accepting the terms and conditions.
6. Provide the required details and click "Submit."
7. You will receive your TMDB API key.

Recreate the Docker image with your API key:

```bash
docker build --build-arg TMDB_V3_API_KEY=<your-api-key> -t netflix .
```

## Phase 2: Security

### Install SonarQube and Trivy

Install SonarQube:

```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```

Access SonarQube at `publicIP:9000` (default username & password is admin)

Install Trivy:

```bash
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
```

To scan an image using Trivy:

```bash
trivy image <imageid>
```

### Integrate SonarQube and Configure
- Integrate SonarQube with your CI/CD pipeline.
- Configure SonarQube to analyze code for quality and security issues.

## Phase 3: CI/CD Setup

### Install Jenkins for Automation

Install Java:

```bash
sudo apt update
sudo apt install fontconfig openjdk-17-jre
java -version
```

Install Jenkins:

```bash
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins
```

Access Jenkins in a web browser using the public IP of your EC2 instance: `publicIp:8080`

### Install Necessary Plugins in Jenkins
Go to Manage Jenkins → Plugins → Available Plugins and install:
- Eclipse Temurin Installer
- SonarQube Scanner
- NodeJs Plugin
- Email Extension Plugin

### Configure Java and Nodejs in Global Tool Configuration
Go to Manage Jenkins → Tools → Install JDK(17) and NodeJs(16) → Click on Apply and Save

### SonarQube Configuration
1. Create a token in SonarQube
2. Add the token to Jenkins (Manage Jenkins → Credentials → Add Secret Text)
3. Configure SonarQube server in Jenkins (Configure System)
4. Install SonarScanner in Jenkins (Global Tool Configuration)
5. Create a Jenkins webhook

### Create a CI/CD Pipeline in Jenkins
Create a new pipeline job and use the following pipeline script:

```groovy
pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/N4si/DevSecOps-Project.git'
            }
        }
        stage("Sonarqube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix'''
                }
            }
        }
        stage("quality gate") {
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
    }
}
```

### Install Dependency-Check and Docker Tools in Jenkins
1. Install OWASP Dependency-Check Plugin
2. Configure Dependency-Check Tool
3. Install Docker Tools and Docker Plugins
4. Add DockerHub Credentials

Update your Jenkins pipeline to include these new stages:

```groovy
pipeline{
    agent any
    tools{
        jdk 'jdk17'
        nodejs 'node16'
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
                git branch: 'main', url: 'https://github.com/N4si/DevSecOps-Project.git'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix '''
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
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){   
                       sh "docker build --build-arg TMDB_V3_API_KEY=<yourapikey> -t netflix ."
                       sh "docker tag netflix nasi101/netflix:latest "
                       sh "docker push nasi101/netflix:latest "
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image nasi101/netflix:latest > trivyimage.txt" 
            }
        }
        stage('Deploy to container'){
            steps{
                sh 'docker run -d --name netflix -p 8081:80 nasi101/netflix:latest'
            }
        }
    }
}
```

If you get a docker login failed error:

```bash
sudo su
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

## Phase 4: Monitoring

### Install Prometheus

1. Create a dedicated Linux user for Prometheus:
   ```bash
   sudo useradd --system --no-create-home --shell /bin/false prometheus
   wget https://github.com/prometheus/prometheus/releases/download/v2.47.1/prometheus-2.47.1.linux-amd64.tar.gz
   ```

2. Extract Prometheus files, move them, and create directories:
   ```bash
   tar -xvf prometheus-2.47.1.linux-amd64.tar.gz
   cd prometheus-2.47.1.linux-amd64/
   sudo mkdir -p /data /etc/prometheus
   sudo mv prometheus promtool /usr/local/bin/
   sudo mv consoles/ console_libraries/ /etc/prometheus/
   sudo mv prometheus.yml /etc/prometheus/prometheus.yml
   ```

3. Set ownership for directories:
   ```bash
   sudo chown -R prometheus:prometheus /etc/prometheus/ /data/
   ```

4. Create a systemd unit configuration file for Prometheus:
   ```bash
   sudo nano /etc/systemd/system/prometheus.service
   ```

   Add the following content:

   ```ini
   [Unit]
   Description=Prometheus
   Wants=network-online.target
   After=network-online.target

   StartLimitIntervalSec=500
   StartLimitBurst=5

   [Service]
   User=prometheus
   Group=prometheus
   Type=simple
   Restart=on-failure
   RestartSec=5s
   ExecStart=/usr/local/bin/prometheus \
     --config.file=/etc/prometheus/prometheus.yml \
     --storage.tsdb.path=/data \
     --web.console.templates=/etc/prometheus/consoles \
     --web.console.libraries=/etc/prometheus/console_libraries \
     --web.listen-address=0.0.0.0:9090 \
     --web.enable-lifecycle

   [Install]
   WantedBy=multi-user.target
   ```

5. Enable and start Prometheus:
   ```bash
   sudo systemctl enable prometheus
   sudo systemctl start prometheus
   ```

6. Verify Prometheus's status:
   ```bash
   sudo systemctl status prometheus
   ```

7. Access Prometheus in a web browser: `http://<your-server-ip>:9090`

### Install Node Exporter

1. Create a system user for Node Exporter and download it:
   ```bash
   sudo useradd --system --no-create-home --shell /bin/false node_exporter
   wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
   ```

2. Extract Node Exporter files, move the binary, and clean up:
   ```bash
   tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz
   sudo mv node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/
   rm -rf node_exporter*
   ```

3. Create a systemd unit configuration file for Node Exporter:
   ```bash
   sudo nano /etc/systemd/system/node_exporter.service
   ```

   Add the following content:

   ```ini
   [Unit]
   Description=Node Exporter
   Wants=network-online.target
   After=network-online.target

   StartLimitIntervalSec=500
   StartLimitBurst=5

   [Service]
   User=node_exporter
   Group=node_exporter
   Type=simple
   Restart=on-failure
   RestartSec=5s
   ExecStart=/usr/local/bin/node_exporter --collector.logind

   [Install]
   WantedBy=multi-user.target
   ```

4. Enable and start Node Exporter:
   ```bash
   sudo systemctl enable node_exporter
   sudo systemctl start node_exporter
   ```

5. Verify the Node Exporter's status:
   ```bash
   sudo systemctl status node_exporter
   ```

### Configure Prometheus

1. Update the prometheus.yml file:

   ```yaml
   global:
     scrape_interval: 15s

   scrape_configs:
     - job_name: 'node_exporter'
       static_configs:
         - targets: ['localhost:9100']

     - job_name: 'jenkins'
       metrics_path: '/prometheus'
       static_configs:
         - targets: ['<your-jenkins-ip>:<your-jenkins-port>']
   ```

2. Check the validity of the configuration file:
   ```bash
   promtool check config /etc/prometheus/prometheus.yml
   ```

3. Reload the Prometheus configuration:
   ```bash
   curl -X POST http://localhost:9090/-/reload
   ```

4. Access Prometheus targets at: `http://<your-prometheus-ip>:9090/targets`

### Install Grafana

1. Install dependencies:
   ```bash
   sudo apt-get update
   sudo apt-get install -y apt-transport-https software-properties-common
   ```

2. Add the GPG key for Grafana:
   ```bash
   wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
   ```

3. Add the repository for Grafana stable releases:
   ```bash
   echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
   ```

4. Update and install Grafana:
   ```bash
   sudo apt-get update
   sudo apt-get -y install grafana
   ```

5. Enable and start Grafana service:
   ```bash
   sudo systemctl enable grafana-server
   sudo systemctl start grafana-server
   ```

6. Check Grafana status:
   ```bash
   sudo systemctl status grafana-server
   ```

7. Access Grafana web interface:
   - Open a web browser and navigate to: `http://<your-server-ip>:3000`
   - Default login: username "admin", password "admin"
   - Change the default password when prompted.

8. Add Prometheus data source:
   - Click on the gear icon (⚙️) in the left sidebar
   - Select "Data Sources"
   - Click on "Add data source"
   - Choose "Prometheus"
   - Set the "URL" to `http://localhost:9090`
   - Click "Save & Test"

9. Import a dashboard:
   - Click on the "+"
