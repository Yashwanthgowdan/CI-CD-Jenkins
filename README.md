DevOps pipeline: from creating a Flask app, putting it in GitHub, writing a Dockerfile, and building a Jenkins pipeline that will:

Build a Docker image
Push it to Docker Hub
SSH into an EC2 instance and deploy (run the container)

🧱 Step-by-Step Guide
🧪 Step 1: Create a Minimal Flask App

Create a project like this:
flask-docker-pipeline/
├── app.py
├── requirements.txt
├── Dockerfile
├── .dockerignore
└── Jenkinsfile

app.py:
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello():
    return "Hello from Flask + Docker + Jenkins!"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
    
requirements.txt
flask==2.3.3

Step 2: Add Dockerfile
FROM python:3.10-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "app.py"]

.dockerignore
__pycache__
*.pyc
*.pyo

Step 3: Push Project to GitHub
Create a new GitHub repo (e.g., flask-docker-jenkins)

Step 4: Set Up Docker Hub
Create a Docker Hub account
Create a new public/private repository (e.g., yourname/flask-demo)
Save your Docker Hub username and access token/password for Jenkins

Step 5: Jenkins Pipeline (Jenkinsfile)
Create this Jenkinsfile in your repo:
pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-creds') // Jenkins credentials ID
        DOCKER_IMAGE = 'yourdockerhubusername/flask-demo'
        EC2_HOST = 'ec2-user@your-ec2-public-ip'
        EC2_KEY = credentials('ec2-key') // SSH private key in Jenkins credentials
    }

    stages {
        stage('Checkout Code') {
            steps {
                git url: 'https://github.com/YOUR_USERNAME/flask-docker-jenkins.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${DOCKER_IMAGE}:latest")
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-creds') {
                        docker.image("${DOCKER_IMAGE}:latest").push()
                    }
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                script {
                    // Pull and run the container on EC2 using SSH
                    sshagent(credentials: ['ec2-key']) {
                        sh """
                        ssh -o StrictHostKeyChecking=no ${EC2_HOST} '
                          docker pull ${DOCKER_IMAGE}:latest &&
                          docker stop flask-app || true &&
                          docker rm flask-app || true &&
                          docker run -d -p 80:5000 --name flask-app ${DOCKER_IMAGE}:latest
                        '
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Deployment successful!"
        }
        failure {
            echo "Deployment failed!"
        }
    }
}

Step 6: Set Up Jenkins Credentials
Go to Jenkins > Manage Jenkins > Credentials

Add: Docker Hub credentials (username/password) as dockerhub-creds

SSH private key for EC2 as ec2-key

Step 7: EC2 Setup
On your EC2 (Amazon Linux / Ubuntu):

sudo yum update -y || sudo apt update -y
sudo yum install docker -y || sudo apt install docker.io -y
sudo systemctl start docker
sudo usermod -aG docker ec2-user
Also allow port 80 in the EC2 Security Group to access the app.

Step 8: Trigger Jenkins Pipeline
Create a new Jenkins Pipeline project
Set it to pull from your GitHub repo
Click Build Now

Visit http://<your-ec2-public-ip> — you should see:
“Hello from Flask + Docker + Jenkins!”
