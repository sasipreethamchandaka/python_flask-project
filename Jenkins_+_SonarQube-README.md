# Jenkins + SonarQube CI Pipeline for Python Flask (AWS EC2)

This project demonstrates a complete **CI pipeline** for a Python Flask application using:

- GitHub (Source Code)
- Jenkins (CI Automation)
- SonarQube (Static Code Analysis)
- SonarScanner CLI
- AWS EC2 (Single server hosting all services)

---

## Server Architecture

All services run on the SAME EC2 instance

    ec2-user → SSH login user
    root     → Administrator (installs tools)
    sonar    → Runs SonarQube service
    jenkins  → Runs CI pipeline jobs

Important:
Jenkins does NOT read ~/.bashrc or user PATH variables.
Tools must be configured inside Jenkins UI.

---

## Project Repository

Repository used inside pipeline:

    https://github.com/sasipreethamchandaka/python_flask-project

Project structure after adding configuration:

    app.py
    requirements.txt
    sonar-project.properties
    Jenkinsfile

---
## server

    required 4gb ram and 2 cpu's
    dependency: java11 or 17
    req: t2.medium
    port: 9000

STEUP:  sonerQube setup:--
       
    yum list java*
    java11 or 17

    cd /opt/ #switch to opt directory
     sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-8.9.6.50800.zip
     sudo unzip sonarqube-8.9.6.50800.zip

#sonarqube has to run with user only 

     sudo useradd sonar
     sudo chown sonar:sonar sonarqube-8.9.6.50800 -R
     sudo chmod 777 sonarqube-8.9.6.50800 -R


-------------passwd sonar (create password)
    
    su - sonar   (log in as sonar)
    password:--

    sh /opt/sonarqube-8.9.6.50800/bin/linux-x86-64/sonar.sh start
    sh /opt/sonarqube-8.9.6.50800/bin/linux-x86-64/sonar.sh status

 #to access 
        
    public-ip:9000
    user=admin & password=admin

note : after login we need to change our existing password to custom password 
---------------------------------------------------------------------

    inside sonerqube got to project and create project then 
     select language and jenerate token  (language--java) 

--------------------------------- other programming languages process------------------------------------

## Install SonarScanner (System Wide)   according to requirements 


Download and extract:

    wget https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-4.8.0.2856-linux.zip
  
    unzip sonar-scanner-cli-4.8.0.2856-linux.zip

Move to global directory:

    sudo mv sonar-scanner-4.8.0.2856-linux /opt/sonar-scanner
    sudo chmod -R 755 /opt/sonar-scanner
    sudo chown -R root:root /opt/sonar-scanner
    
Verify sonar can execute scanner:

    sudo -u sonar /opt/sonar-scanner/bin/sonar-scanner -v

    Expected:

       SonarScanner 4.8.0.2856

 Set up environment variables:
    
     Add the following lines to your .bashrc, .zshrc, or .profile file: (open vi and other cmd)
      [sonar@ip-172-31-28-135 ~]$ vi .bashrc
       add in the last -----------export PATH=$PATH:/opt/sonar-scanner/bin
                              source ~/.bashrc    ## Verify the installation:
                              sonar-scanner --version


## Go to root dir 
      yum install git -y
## clone project inside sonar
    [sonar@ip-172-31-28-135 ~]$ git clone https://github.com/sasipreethamchandaka/python_flask-project.git
      cd python_flask-project

 -------- project folder inside add file vi sonar-project.properties---------------
     paste this code---

    sonar.projectKey=flask-app
    sonar.projectName=flask-app
    sonar.projectVersion=1.0

    sonar.sources=.
    sonar.sourceEncoding=UTF-8

    sonar.python.version=3

    sonar.exclusions=venv/**,__pycache__/**,tests/**
    sonar.login=2baa0e76d05c1c664a4e5f6a4fbf8b9799be1027  (sonerqube jenerate security token)
    
Push to GitHub:

    git add sonar-project.properties
    git commit -m "Add SonarQube configuration"
    git push main origin
    
    
 ---------foder inside paste sonar project token 
      
    Ex: 
      sonar-scanner \
     -Dsonar.projectKey=python1 \
     -Dsonar.sources=. \
     -Dsonar.host.url=http://43.204.112.71:9000 \
     -Dsonar.login=token   
     

# configure wuith jenkins 

  install jenkins root ~ dir 
#------------jenkins install-------------

    sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
    sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
    sudo yum install jenkins -y
    sudo systemctl enable jenkins
    sudo systemctl start jenkins

long in 

    http:<public-key>:8080

      
---

## Configure SonarQube Server in Jenkins
--Required plugins--

          SonarQube scanner for Jenkins
          pipeline stage view
          blue ocean

Navigate:

    Manage Jenkins → Configure System → SonarQube servers

Add server:

    Name: sonar
    Server URL: http://public-IP:9000
    Token: generated from SonarQube → My Account → Security → Generate Token

---

## Configure SonarScanner in Jenkins

Navigate:

    Manage Jenkins → Global Tool Configuration → SonarQube Scanner

Add configuration:

    Name: sonar-scanner
    Install automatically: unchecked
    SONAR_RUNNER_HOME: /opt/sonar-scanner

## SonarQube Project Configuration  (second format)

Create file:

    sonar-project.properties

Content:

    sonar.projectKey=flask-app
    sonar.projectName=flask-app
    sonar.projectVersion=1.0
    sonar.sources=.
    sonar.sourceEncoding=UTF-8
    sonar.python.version=3
    sonar.exclusions=venv/**,__pycache__/**,tests/**

Push to GitHub:

    git add sonar-project.properties
    git commit -m "Add SonarQube configuration"
    git push origin main                 

---

for this foramt use this pipeline ------
## Jenkins Pipeline (Jenkinsfile)

The pipeline uses the repository URL directly:

    https://github.com/CloudTechDevOps/docker_python_flask-project.git

Pipeline definition:


    pipeline {
        agent any

        environment {
            SONARQUBE = 'sonar'
        }

        stages {

            stage('Checkout Code') {
                steps {
                    git branch: 'main', url: 'https://github.com/CloudTechDevOps/docker_python_flask-project.git'
                }
            }

            stage('Install Dependencies') {
                steps {
                    sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install -r requirements.txt || true
                    '''
                }
            }

            stage('Run Tests') {
                steps {
                    sh '''
                    . venv/bin/activate
                    pip install pytest || true
                    pytest || true
                    '''
                }
            }

            stage('SonarQube Analysis') {
                steps {
                    script {
                        def scannerHome = tool 'sonar-scanner'
                        withSonarQubeEnv('sonar') {
                            sh "${scannerHome}/bin/sonar-scanner"
                        }
                    }
                }
            }
        }
    }

---

------Running the Pipeline

Open Jenkins:

    http://EC2-PUBLIC-IP:8080

Trigger:

    Pipeline → Build Now

---

## Expected Successful Output

    Project root configuration file: sonar-project.properties
    SonarScanner 4.8.0.2856
    Analyzing on SonarQube server
    ANALYSIS SUCCESSFUL
    Dashboard: http://PRIVATE-IP:9000/dashboard?id=flask-app

Open SonarQube:

    http://EC2-PUBLIC-IP:9000


##  otherwise use this pipeline    for (firt first format)

          
    pipeline {
       agent any

       environment {
         SONARQUBE = 'sonarji'  // Your Jenkins SonarQube server name
      }

      stages {

          stage('Checkout Code') {
            steps {
                  git branch: 'main', url: 'https://github.com/CloudTechDevOps/docker_python_flask-project.git'
              }
          }

          stage('Install Dependencies') {
            steps {
                  sh '''
                  python3 -m venv venv
                  . venv/bin/activate
                  pip install -r requirements.txt || true
                 '''
              }
          }

          stage('Run Tests') {
              steps {
                  sh '''
                  . venv/bin/activate
                  pip install pytest || true
                  pytest || true
                  '''
              }
          }

         stage('SonarQube Analysis') {
              steps {
                  script {
                      def scannerHome = tool 'sonar-scanner'
                      withSonarQubeEnv('sonarji') {
                          sh """
                          ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=flask-app \
                            -Dsonar.projectName=flask-app \
                            -Dsonar.projectVersion=1.0 \
                            -Dsonar.sources=. \
                            -Dsonar.exclusions=venv/**,__pycache__/**,tests/** \
                            -Dsonar.host.url=$SONAR_HOST_URL \
                            -Dsonar.login=$SONAR_AUTH_TOKEN
                          """
                       }
                    }
                }
            }
        }
     }



Your project will appear with metrics:

    Bugs
    Vulnerabilities
    Code Smells
    Coverage
    Duplications

---



## Common Errors & Solutions

### sonar-scanner: command not found
Cause:
Scanner installed in user home instead of global path

Fix:

    Move to /opt/sonar-scanner
    Configure in Jenkins Global Tool Configuration

---

### Use a tool from predefined Tool Installation
Cause:
Name mismatch

Fix:

    Jenkins tool name MUST match:
    tool 'sonar-scanner'

---

### Missing sonar.projectKey
Cause:
sonar-project.properties not created

Fix:
Create config file in repo root

---

### Permission denied sudoers
Cause:
Installing tools as sonar user

Fix:
Install software as root
Run services as service users

---

### Jenkins ignores PATH or .bashrc
Cause:
Jenkins runs as daemon

Fix:
Never rely on environment variables
Always configure tools in Jenkins UI

---

## Final Workflow

    Developer pushes code → GitHub
    Jenkins pulls code from https://github.com/CloudTechDevOps/docker_python_flask-project.git
    Creates Python virtual environment
    Installs dependencies
    Runs tests
    Executes SonarScanner
    Sends report to SonarQube
    SonarQube dashboard displays code quality

You now have a fully working CI quality pipeline 🚀
