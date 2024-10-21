pipeline {
    agent any
    environment {
        IMAGE = "neathtan/clone_gradle"
        FILE_NAME = "grandle.zip"
        DIR_UNZIP = "grandle17"
        DOCKER_IMAGE = "${IMAGE}:${BUILD_NUMBER}"
        DOCKER_CONTAINER = "springbootG_jenkins"
        DOCKER_CREDENTIALS_ID = "dockertoken"
        GIT_BRANCH = "master"
        GIT_MANIFEST_REPO = "https://github.com/WexleyTan/unzip_gradle_manifest.git"
        MANIFEST_REPO = "unzip_gradle_manifest"
        MANIFEST_FILE_PATH = "deployment.yaml"
        GIT_CREDENTIALS_ID = 'git_pass'
    }

    stages {

        stage('Unzip File') {
            steps {
                script {
                    echo "Checking if the file ${FILE_NAME} exists and unzipping it if present..."
                    sh """
                        if [ -f '${FILE_NAME}' ]; then
                            echo "Removing existing directory ${DIR_UNZIP}..."
                            rm -rf ${DIR_UNZIP}
                            echo "Unzipping the file..."
                            unzip -o '${FILE_NAME}'
                        else
                            echo "'${FILE_NAME}' does not exist."
                            exit 1
                        fi
                    """
                }
            }
        }

        stage('Create Dockerfile') {
            steps {
                script {
                    echo "Creating Dockerfile..."
                    writeFile file: 'Dockerfile', text: '''
                    FROM eclipse-temurin:17-jdk AS build
                    WORKDIR /usr/app/
                    COPY . .
                    RUN chmod +x gradlew
                    RUN ./gradlew bootJar

                    FROM eclipse-temurin:17-jdk
                    ENV JAR_NAME=app.jar
                    ENV APP_HOME=/usr/app/
                    WORKDIR $APP_HOME
                    COPY --from=build $APP_HOME/build/libs/*.jar app.jar
                    EXPOSE 8080
                    ENTRYPOINT ["java", "-jar", "/usr/app/app.jar"]
                    '''
                }
            }
        }


        stage('Build and Push Docker Image') {
            steps {
                script {
                    echo "Building Docker image..."
                    
                    dir("${DIR_UNZIP}") {
                        sh "cp ../Dockerfile ."
                        sh "sed -i 's/languageVersion = JavaLanguageVersion.of([0-9]*)/languageVersion = JavaLanguageVersion.of(17)/' build.gradle"
                        sh "docker build -t ${DOCKER_IMAGE} ."  
                    }
                    sh "docker images | grep -i ${IMAGE}"

                    echo "Logging in to Docker Hub using Jenkins credentials..."
                    withCredentials([usernamePassword(credentialsId: DOCKER_CREDENTIALS_ID, passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER')]) {
                        sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
                    }

                    echo "Pushing the image to Docker Hub..."
                    sh "docker push ${DOCKER_IMAGE}"
                }
            }
        }
        stage("Cloning the Manifest File") {
            steps {
                script {
                    echo "Checking if the manifest repository exists and removing it if necessary..."
                    sh """
                        if [ -d "${MANIFEST_REPO}" ]; then
                            echo "Directory ${MANIFEST_REPO} exists, removing it..."
                            rm -rf ${MANIFEST_REPO}
                        fi
                    """
                    
                    echo "Cloning the manifest repository..."
                    sh "git clone -b ${GIT_BRANCH} ${GIT_MANIFEST_REPO} ${MANIFEST_REPO}"  
                }
            }
        }
        
        stage("Updating the Manifest File") {
            steps {
                script {
                    echo "Updating the image in the deployment manifest..."
                    dir("${MANIFEST_REPO}") {
                        sh """
                            sed -i 's|image: ${IMAGE}:.*|image: ${DOCKER_IMAGE}|' ${MANIFEST_FILE_PATH}
                            echo "Updated deployment file:"
                        """
                        
                        echo "Committing and pushing changes to the manifest repository..."
                        withCredentials([usernamePassword(credentialsId: GIT_CREDENTIALS_ID, passwordVariable: 'GIT_PASS', usernameVariable: 'GIT_USER')]) {
                            sh """
                                git config --global user.name "WexleyTan"
                                git config --global user.email "neathtan1402@gmail.com"
                                git add ${MANIFEST_FILE_PATH}
                                git commit -m "Update image to ${DOCKER_IMAGE}"
                                git push https://${GIT_USER}:${GIT_PASS}@github.com/WexleyTan/unzip_gradle_manifest ${GIT_BRANCH}
                            """
                        }
                    }
                }
            }
        }
    }
}