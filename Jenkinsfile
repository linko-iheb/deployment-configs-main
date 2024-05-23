pipeline {
    agent any

    environment {
        DOCKER_CREDENTIALS_ID = 'dockerhub-credentials' // ID of Docker Hub credentials stored in Jenkins
        DOCKER_HUB_PREFIX = 'houbalinko/' // Default Docker Hub repository prefix
        DOCKER_COMPOSE_FILE = 'docker-compose.yml' // File name for Docker Compose stack
    }

    parameters {
        string(name: 'GITHUB_REPO', defaultValue: '', description: 'URL of the user\'s GitHub repository')
        string(name: 'IMAGE_NAME', defaultValue: 'example-app', description: 'Name of the Docker image (without repository prefix)')
        string(name: 'APP_ID', defaultValue: 'myAppId', description: 'Application ID for Parse Server')
        string(name: 'MASTER_KEY', defaultValue: 'myMasterKey', description: 'Master Key for Parse Server')
    }

    stages {
        stage('Clone User Repository') {
            steps {
                script {
                    echo "Cloning the user repository from ${params.GITHUB_REPO}"
                }
                git url: "${params.GITHUB_REPO}"
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

                        // Update existing environment variables
                        dockerfileContent = dockerfileContent.replaceAll(/ENV APP_ID .*/, "ENV APP_ID ${params.APP_ID}")
                        dockerfileContent = dockerfileContent.replaceAll(/ENV MASTER_KEY .*/, "ENV MASTER_KEY ${params.MASTER_KEY}")
                        dockerfileContent = dockerfileContent.replaceAll(/ENV DATABASE_URI .*/, "ENV DATABASE_URI mongodb://mongodb:27017")

                        // Ensure the new ENV variables are added after the existing ones
                        def envBlock = ""
                        if (!dockerfileContent.contains("ENV MASTER_KEY_IPS")) {
                            envBlock += "ENV MASTER_KEY_IPS \"::/0\"\n"
                        } else {
                            dockerfileContent = dockerfileContent.replaceAll(/ENV MASTER_KEY_IPS .*/, "ENV MASTER_KEY_IPS \"::/0\"")
                        }

                        if (!dockerfileContent.contains("ENV CLOUD_CODE_MAIN")) {
                            envBlock += "ENV CLOUD_CODE_MAIN /parse/cloud/main.js\n"
                        } else {
                            dockerfileContent = dockerfileContent.replaceAll(/# ENV CLOUD_CODE_MAIN .*/, "ENV CLOUD_CODE_MAIN /parse/cloud/main.js")
                        }

                        if (!dockerfileContent.contains("ENV PARSE_SERVER_API_VERSION")) {
                            envBlock += "ENV PARSE_SERVER_API_VERSION 7\n"
                        } else {
                            dockerfileContent = dockerfileContent.replaceAll(/# ENV PARSE_SERVER_API_VERSION .*/, "ENV PARSE_SERVER_API_VERSION 7")
                        }

                        // Insert the new ENV variables after the existing ones
                        dockerfileContent = dockerfileContent.replaceFirst(/(ENV DATABASE_URI .*)/, "\$1\n${envBlock.trim()}")

                        // Remove the '#' character from the commented lines
                        dockerfileContent = dockerfileContent.replaceAll(/# ENV CLOUD_CODE_MAIN .*/, "ENV CLOUD_CODE_MAIN /parse/cloud/main.js")
                        dockerfileContent = dockerfileContent.replaceAll(/# ENV PARSE_MOUNT .*/, "ENV PARSE_MOUNT /parse")

                        writeFile file: 'Dockerfile', text: dockerfileContent
                        echo "Dockerfile updated"
                        sh 'cat Dockerfile'
                    } else {
                        writeFile file: 'Dockerfile', text: """
                        FROM node:latest

                        # Set the working directory to /parse
                        WORKDIR /parse

                        # Copy package.json and package-lock.json files
                        COPY package*.json ./

                        # Install dependencies
                        RUN npm install

                        # Copy the rest of the application code
                        COPY . .

                        # Set environment variables
                        ENV APP_ID=${params.APP_ID}
                        ENV MASTER_KEY=${params.MASTER_KEY}
                        ENV DATABASE_URI=mongodb://mongodb:27017
                        ENV PARSE_MOUNT=/parse
                        ENV CLOUD_CODE_MAIN=/parse/cloud/main.js
                        ENV PARSE_SERVER_API_VERSION=7

                        # Expose port 1337
                        EXPOSE 1337

                        # Start the Parse Server
                        CMD [ "npm", "start" ]
                        """
                        echo "Dockerfile created"
                        sh 'cat Dockerfile'
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

                    def imageName = "${env.DOCKER_HUB_PREFIX}${params.IMAGE_NAME}"
                    // Generate Docker Compose stack file content with volumes and depends_on
                    def dockerComposeStackContent = """
                    version: '3.8'

                    services:
                      parse:
                        image: ${imageName}
                        environment:
                          - APP_ID=${params.APP_ID}
                          - MASTER_KEY=${params.MASTER_KEY}
                          - DATABASE_URI=mongodb://mongodb:27017
                          - PARSE_MOUNT=/parse
                          - CLOUD_CODE_MAIN=/parse/cloud/main.js
                          - PARSE_SERVER_API_VERSION=7
                        ports:
                          - "1337:1337"
                        volumes:
                          - parse_data:/parse
                        depends_on:
                          - mongodb
                        networks:
                          - parse_server_network

                      parse-dashboard:
                        image: parseplatform/parse-dashboard
                        environment:
                          - PARSE_DASHBOARD_SERVER_URL=http://localhost:1337/parse
                          - PARSE_DASHBOARD_APP_ID=${params.APP_ID}
                          - PARSE_DASHBOARD_MASTER_KEY=${params.MASTER_KEY}
                          - PARSE_DASHBOARD_APP_NAME=myAppName
                          - PARSE_DASHBOARD_USER_ID=username
                          - PARSE_DASHBOARD_USER_PASSWORD=password
                          - PARSE_DASHBOARD_ALLOW_INSECURE_HTTP=true
                        ports:
                          - "4040:4040"
                        networks:
                          - parse_server_network
                        depends_on:
                          - parse

                      mongodb:
                        image: mongo:latest
                        networks:
                          - parse_server_network

                    networks:
                      parse_server_network:
                    volumes:
                      parse_data:
                      mongo_data:
                    """
                    // Write Docker Compose stack file
                    writeFile file: "${env.DOCKER_COMPOSE_FILE}", text: dockerComposeStackContent
                }
                echo "Docker Compose stack file created successfully"
                sh "cat ${env.DOCKER_COMPOSE_FILE}"
            }
        }

        
        stage('Deploy Docker Compose Stack') {
            steps {
                script {
                    // Add a delay to ensure volume creation is completed
                    sleep(time: 10, unit: 'SECONDS')

                    // Deploy the Docker Compose stack
                    sh "docker stack deploy -c ${env.DOCKER_COMPOSE_FILE} parse-stack"
                }
                echo "Docker Compose stack deployed successfully"
            }
        }

    

        stage('Cleanup') {
            steps {
                script {
                    // Delete Dockerfile and Docker Compose file
                    sh "rm Dockerfile ${env.DOCKER_COMPOSE_FILE}"
                    sh "rm -rf ${env.GITHUB_REPO}"
                }
                echo "Dockerfile and Docker Compose file deleted successfully"
            }
        }
    }
}
