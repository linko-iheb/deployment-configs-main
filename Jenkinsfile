pipeline {
    agent any

    environment {
        DOCKER_CREDENTIALS_ID = 'dockerhub-credentials' // ID of Docker Hub credentials stored in Jenkins
        DOCKER_HUB_PREFIX = 'houbalinko/' // Default Docker Hub repository prefix
    }

    parameters {
        string(name: 'USER_REPO', defaultValue: 'https://github.com/example-user/example-repo.git', description: 'URL of the user\'s GitHub repository')
        string(name: 'IMAGE_NAME', defaultValue: 'example-app', description: 'Name of the Docker image (without repository prefix)')
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

        stage('Docker Build') {
            steps {
                script {
                    def imageName = "${env.DOCKER_HUB_PREFIX}${params.IMAGE_NAME}"
                    // Build the Docker image
                    sh "docker build -t ${imageName} ."
                }
                echo "Docker build completed"
            }
        }
    }
}
