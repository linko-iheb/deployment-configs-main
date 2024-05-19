pipeline {
    agent any

    environment {
        DOCKER_CREDENTIALS_ID = 'dockerhub-credentials' // ID of Docker Hub credentials stored in Jenkins
    }

    parameters {
        string(name: 'USER_REPO', defaultValue: '', description: 'URL of the user\'s GitHub repository')
        string(name: 'DOCKER_IMAGE', defaultValue: 'your-dockerhub-username/user-app:latest', description: 'Docker image name')
    }

    stages {
        stage('Clone User Repository') {
            steps {
                script {
                    echo "Cloning the user repository from ${params.USER_REPO}"
                }
                git url: "${params.USER_REPO}"
                script {
                    echo "Repository successfully cloned"
                    echo "Listing the contents of the workspace:"
                    sh 'ls -la'
                }
            }
        }

        stage('Create Dockerfile if Missing') {
            steps {
                script {
                    if (!fileExists('Dockerfile')) {
                        writeFile file: 'Dockerfile', text: """
                        FROM node:14
                        WORKDIR /usr/src/app
                        COPY package*.json ./
                        RUN npm install
                        COPY . .
                        EXPOSE 8080
                        CMD [ "node", "index.js" ]
                        """
                        echo "Dockerfile created"
                    } else {
                        echo "Dockerfile already exists"
                    }
                }
            }
        }

       
    }
}
