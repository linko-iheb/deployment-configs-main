pipeline {
    agent any

    environment {
        DOCKER_CREDENTIALS_ID = 'dockerhub-credentials' // ID of Docker Hub credentials stored in Jenkins
        DOCKER_HUB_PREFIX = 'houbalinko/' // Default Docker Hub repository prefix
    }

    parameters {
        string(name: 'USER_REPO', defaultValue: 'https://github.com/example-user/example-repo.git', description: 'URL of the user\'s GitHub repository')
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

                        # Optional (default : 'parse/cloud/main.js')
                        # ENV CLOUD_CODE_MAIN cloudCodePath

                        # Optional (default : '/parse')
                        # ENV PARSE_MOUNT mountPath

                        EXPOSE 1337

                        # Uncomment if you want to access cloud code outside of your container
                        # A main.js file must be present, if not Parse will not start

                        # VOLUME /parse/cloud               

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
}
