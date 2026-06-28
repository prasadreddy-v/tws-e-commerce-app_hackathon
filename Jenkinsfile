pipeline {
    agent any

    environment {
        DOCKER_IMAGE_NAME = "prasadhandsondock/easyshop-app"
        DOCKER_MIGRATION_IMAGE_NAME = "prasadhandsondock/easyshop-migration"
        IMAGE_TAG = "${BUILD_NUMBER}"

        GIT_REPO = "https://github.com/prasadreddy-v/tws-e-commerce-app_hackathon.git"
        GIT_BRANCH = "master"
    }

    stages {

        stage('Clean Workspace') {
            steps {
                deleteDir()
            }
        }

        stage('Checkout Source') {
            steps {
                git branch: "${GIT_BRANCH}",
                    credentialsId: "github-credentials",
                    url: "${GIT_REPO}"
            }
        }

        stage('Build Application') {
            steps {
                sh '''
                if [ -f mvnw ]; then
                    chmod +x mvnw
                    ./mvnw clean package -DskipTests
                elif [ -f pom.xml ]; then
                    mvn clean package -DskipTests
                else
                    echo "No Maven project found."
                fi
                '''
            }
        }

        stage('Run Unit Tests') {
            steps {
                sh '''
                if [ -f mvnw ]; then
                    ./mvnw test
                elif [ -f pom.xml ]; then
                    mvn test
                else
                    echo "No Maven project found."
                fi
                '''
            }
        }

        stage('Build Docker Images') {
            parallel {

                stage('Main Image') {
                    steps {
                        sh """
                        docker build \
                        -t ${DOCKER_IMAGE_NAME}:${IMAGE_TAG} \
                        -f Dockerfile .
                        """
                    }
                }

                stage('Migration Image') {
                    steps {
                        sh """
                        docker build \
                        -t ${DOCKER_MIGRATION_IMAGE_NAME}:${IMAGE_TAG} \
                        -f scripts/Dockerfile.migration .
                        """
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh """
                mkdir -p trivy-reports

                trivy image \
                --severity HIGH,CRITICAL \
                --format table \
                --output trivy-reports/app-image.txt \
                ${DOCKER_IMAGE_NAME}:${IMAGE_TAG}

                trivy image \
                --severity HIGH,CRITICAL \
                --format table \
                --output trivy-reports/migration-image.txt \
                ${DOCKER_MIGRATION_IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }

        stage('Push Docker Images') {
            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'docker-hub-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {

                    sh """
                    echo "\$DOCKER_PASS" | docker login -u "\$DOCKER_USER" --password-stdin

                    docker push ${DOCKER_IMAGE_NAME}:${IMAGE_TAG}
                    docker push ${DOCKER_MIGRATION_IMAGE_NAME}:${IMAGE_TAG}

                    docker logout
                    """
                }
            }
        }

        stage('Update Kubernetes Manifest') {
            steps {

                sh """
                sed -i 's|image: ${DOCKER_IMAGE_NAME}:.*|image: ${DOCKER_IMAGE_NAME}:${IMAGE_TAG}|' kubernetes/deployment.yaml || true
                """

                withCredentials([
                    usernamePassword(
                        credentialsId: 'github-credentials',
                        usernameVariable: 'GIT_USER',
                        passwordVariable: 'GIT_TOKEN'
                    )
                ]) {

                    sh """
                    git config user.name "Jenkins CI"
                    git config user.email "misc.lucky66@gmail.com"

                    git add kubernetes/

                    git commit -m "Updated image tag to ${IMAGE_TAG}" || true

                    git push https://\$GIT_USER:\$GIT_TOKEN@github.com/prasadreddy-v/tws-e-commerce-app_hackathon.git HEAD:master
                    """
                }
            }
        }

    }

    post {

        always {
            archiveArtifacts artifacts: 'trivy-reports/*', allowEmptyArchive: true
        }

        success {
            echo "Pipeline completed successfully."
        }

        failure {
            echo "Pipeline failed."
        }
    }
}
