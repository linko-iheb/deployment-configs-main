pipeline {
    agent any

    environment {
        DOCKER_CREDENTIALS_ID = 'dockerhub-credentials' // ID of Docker Hub credentials stored in Jenkins
        DOCKER_HUB_PREFIX = 'houbalinko/' // Default Docker Hub repository prefix
        DOCKER_COMPOSE_FILE = 'docker-compose.yml' // File name for Docker Compose stack
    }

    parameters {
        string(name: 'USER_REPO', defaultValue: 'https://github.com/parse-community/parse-server-example', description: 'URL of the user\'s GitHub repository')
        string(name: 'IMAGE_NAME', defaultValue: 'example-app', description: 'Name of the Docker image (without repository prefix)')
        string(name: 'APP_ID', defaultValue: 'yourAppId', description: 'Application ID for Parse Server')
        string(name: 'MASTER_KEY', defaultValue: 'yourMasterKey', description: 'Master Key for Parse Server')
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

        stage('Create or Update Dockerfile') {
            steps {
                script {
                    if (fileExists('Dockerfile')) {
                        echo "Dockerfile already exists, updating with parameters"
                        def dockerfileContent = readFile('Dockerfile')
                        dockerfileContent = dockerfileContent.replaceAll(/ENV APP_ID .*/, "ENV APP_ID ${params.APP_ID}")
                        dockerfileContent = dockerfileContent.replaceAll(/ENV MASTER_KEY .*/, "ENV MASTER_KEY ${params.MASTER_KEY}")
                        dockerfileContent = dockerfileContent.replaceAll(/ENV DATABASE_URI .*/, "ENV DATABASE_URI mongodb://mongo:27017")
                        writeFile file: 'Dockerfile', text: dockerfileContent
                        echo "Dockerfile updated"
                    } else {
                        writeFile file: 'Dockerfile', text: """
                        FROM node:latest

                        RUN mkdir parse

                        ADD . /parse
                        WORKDIR /parse
                        RUN npm install

                        ENV APP_ID=${params.APP_ID}
                        ENV MASTER_KEY=${params.MASTER_KEY}
                        ENV DATABASE_URI=mongodb://mongo:27017

                        EXPOSE 1337

                        CMD [ "npm", "start" ]
                        """
                        echo "Dockerfile created"
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
                echo "Docker build completed successfully"
            }
        }

        stage('Create Docker Compose Stack') {
            steps {
                script {
                    // Check if the parse_data volume already exists
                    def parseVolumeCheck = sh(script: 'docker volume ls -q -f name=parse_data', returnStdout: true).trim()
                    if (parseVolumeCheck.isEmpty()) {
                        // If the volume doesn't exist, create it
                        sh "docker volume create parse_data"
                        echo "parse_data volume created"
                    } else {
                        echo "parse_data volume already exists"
                    }

                    // Check if the mongo_data volume already exists
                    def mongoVolumeCheck = sh(script: 'docker volume ls -q -f name=mongo_data', returnStdout: true).trim()
                    if (mongoVolumeCheck.isEmpty()) {
                        // If the volume doesn't exist, create it
                        sh "docker volume create mongo_data"
                        echo "mongo_data volume created"
                    } else {
                        echo "mongo_data volume already exists"
                    }

                    // Generate Docker Compose stack file content with volumes
                    def dockerComposeStackContent = """
                        version: '3.8'
                        services:
                          parse:
                            image: ${params.IMAGE_NAME}
                            environment:
                              APP_ID: ${params.APP_ID}
                              MASTER_KEY: ${params.MASTER_KEY}
                              DATABASE_URI: mongodb://mongo:27017
                            ports:
                              - "1337:1337"
                            volumes:
                              - parse_data:/parse
                          mongo:
                            image: mongo:latest
                            volumes:
                              - mongo_data:/data/db
                    """
                    // Write Docker Compose stack file
                    writeFile file: "${env.DOCKER_COMPOSE_FILE}", text: dockerComposeStackContent
                }
                echo "Docker Compose stack file created successfully"
            }
        }

        stage('Deploy Docker Compose Stack') {
            steps {
                script {
                    // Deploy the Docker Compose stack
                    sh "docker stack deploy -c ${env.DOCKER_COMPOSE_FILE} parse-stack"
                }
                echo "Docker Compose stack deployed successfully"
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    def imageName = "${env.DOCKER_HUB_PREFIX}${params.IMAGE_NAME}"
                    // Login to Docker Hub using Jenkins credentials
                    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: "${env.DOCKER_CREDENTIALS_ID}", usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD']]) {
                        sh "docker login -u ${DOCKER_USERNAME} -p ${DOCKER_PASSWORD}"
                    }
                    // Push the Docker image to Docker Hub
                    sh "docker push ${imageName}"
                }
                echo "Docker image pushed to Docker Hub successfully"
            }
        }
    }

    post {
        always {
            // Clean up the Docker Compose stack file
            deleteFile "${env.DOCKER_COMPOSE_FILE}"
        }
    }
}
