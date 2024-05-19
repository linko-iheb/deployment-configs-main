pipeline {
    agent any

    environment {
        DOCKER_CREDENTIALS_ID = 'dockerhub-credentials' // ID of Docker Hub credentials stored in Jenkins
        SWARM_MANAGER_HOST = credentials('swarm-manager-host') // Host of Docker Swarm manager stored in Jenkins
        SWARM_MANAGER_PORT = credentials('swarm-manager-port') // Port of Docker Swarm manager stored in Jenkins
    }

    parameters {
        string(name: 'USER_REPO', defaultValue: '', description: 'URL of the user\'s GitHub repository')
        string(name: 'DOCKER_IMAGE', defaultValue: 'your-dockerhub-username/user-app:latest', description: 'Docker image name')
    }

    stages {
        stage('Clone User Repository') {
            steps {
                git url: "${params.USER_REPO}"
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
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', DOCKER_CREDENTIALS_ID) {
                        def image = docker.build("${params.DOCKER_IMAGE}")
                        image.push()
                    }
                }
            }
        }

        stage('Deploy to Docker Swarm') {
            steps {
                script {
                    sh """
                    export DOCKER_HOST=tcp://${SWARM_MANAGER_HOST}:${SWARM_MANAGER_PORT}
                    docker service update --image ${params.DOCKER_IMAGE} user-app || \
                    docker service create --name user-app --publish 8080:8080 ${params.DOCKER_IMAGE}
                    """
                }
            }
        }
    }
}
