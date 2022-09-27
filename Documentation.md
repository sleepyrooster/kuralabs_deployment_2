<h1>Deplpyment 2 Documentation</h1>

## Purpose of Deployment
Deployment 2 is meant to simulate how a DevOps engineer would use the terminal to:
1) Install the AWS CLI and EB CLI onto an EC2 in the terminal
2) Use Jenkins as a manager to build, test and deploy an application to an elastic beanstalk instance

### Observation
While performing this deployment, the following steps were done to set up the CI/CD pipepline:
1) Started this deployment on a fresh EC2 so I gave it the following requirements:
- The ports which were open on the EC2 was: 
  - SSH (port 22)
  - HTTP (port 80)
  - Port 8080 (Port opened for Jenkins Server)
 
 - Wanted to install Jenkins on the EC2 once it was initalized so in the user data tab the following command was added:
 ```
 # Installing Java JDK
 
 $ sudo apt update
 $ sudo apt install default-jre -y
 
 # Installing Jenkins
 
 $ curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee \
    /usr/share/keyrings/jenkins-keyring.asc > /dev/null
 $ echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
   https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
   /etc/apt/sources.list.d/jenkins.list > /dev/null
 $ sudo apt-get update
 $ sudo apt-get install jenkins
 $ sudo systemctl enable jenkins
 $ sudo systemctl start jenkins
 ```
2) After the EC2 was created, `python-pip` and `python3.10-venv` was installed.
```
$ sudo apt update
$ sudo apt install python3-pip
$ sudo apt install python3.10-venv
```
3) Once python and it dependencies were installed, log into jenkin using the EC2 IP address and port 8080 at the end of the URL. Once logged in
  the plugins was installed and a user was created.
5) Then the Jenkins system user was activated which was created when Jenkins was installed on the EC2.
   ```
   $ sudo passwd jenkins
   $ sudo su - jenkins -s /bin/bash
   ```
6) A Jenkins user was created on the AWS Account using IAM.
- This AWS user was called "EB-User" and was given the "programmatic access" type
- After selecting "Attached Exsisting Policies Directly", the user was given the "administrator access" policy.
- After the user was created the access key and the secret access key was copied for the AWS CLI installation

7) Then the AWS CLI was installed using the following commands:
```
$ curl "https://awscli.amazoncli.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
$ unzip awscliv2.zip
$ sudo ./aws/install
$ aws --version

$ sudo su - jenkins -s /bin/bash

$ aws config
  - set access key ID
  - set secret access key
  - set region: (your region, mine was us-east-1)
  - set output: json
```
8) After the EB CLI was installed
```
$ sudo su - jenkins -s /bin/bash
$ pip install awsecli --upgrade --user
$ eb --version
```
9) Once the AWS and the EB CLIs were installed, the Deployment repo was forked and a personal token was created. When the person token was created, the following permissions was selected:
 - repo
 - admin:repo

10) Then a new multibranch pipeline was created and given the name "url-shortener"
11) The branch source option was selected and the source "Github" was added
- For the creditentials the username is the github account username and the password is the personal token.
- The description of "Github Token" was added. 
- The URL of the forked repo was added to validate the repository
12) Once all other setting was checked, the pipeline was applied and saved. Jenkins then started a build based on the Jenkinsfile.
13) To deploy the application to Elastic Beanstalk using the EB CLI, the following commands was used:
```
$ sudo su - jenkins -s /bin/bash
$ cd workspace/url-shortener-main
$ eb init
```
- The following options were selected when `eb init` ran:
  - The region "us-east-1" was selected
  - The project was selected (which was supposed to be the default option)
  - The latest python version was selected
  - CodeCommit wasn't selected
Then the `eb create` command was ran to create the Elastic Beanstalk
- The following options were selected when `eb create` was ran:
  - The environment name which is used is "url-shortener-main-dev"
  - The next 2 default options were selected 
  - The spot fleet option wasn't selected

14) Once the Elastic Beanstalk was created, a deploy stage was added to the Jenkinsfile
```
pipeline {
  agent any
   stages {
    stage ('Build') {
      steps {
        sh '''#!/bin/bash
        python3 -m venv test3
        source test3/bin/activate
        pip install pip --upgrade
        pip install -r requirements.txt
        export FLASK_APP=application
        flask run &
        '''
     }
   }
    stage ('test') {
      steps {
        sh '''#!/bin/bash
        source test3/bin/activate
        py.test --verbose --junit-xml test-reports/results.xml
        ''' 
      }
    
      post{
        always {
          junit 'test-reports/results.xml'
        }
       
      }
    }
     stage ('Deploy') {
       steps{
         sh '/var/lib/jenkins/.local/bin/eb deploy url-shortener-main-dev'
       }
     }
   
  }
 }
```
15) Once the deploy stage was added, Jenkins scanned the repository and started another build.



### Errors while performing deployment
1) Installing Elastic Beanstalk CLI on the EC2 instance
- After installing the EB CLI to the Jenkins user, I couldn't use `$ eb --version` to check to see the version of the EB CLI
- To fix this I needed to copy the path where the EB CLI was stored and placed it in the `/etc/environment` so that I could use it globally
```
$ sudo nano /etc/environment
```
I copied and paste the `~/.local/bin` path to the `/etc/environment`

2) Unziping the AWS CLI zip to install the AWS CLI
- When installing the AWS CLI I was unable to unzip the file because unzip wasn't installed on my EC2. So I installed it using the following command:
```
$ sudo apt install unzip
```
