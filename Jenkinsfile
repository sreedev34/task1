
pipeline {
    agent any

    environment {
        DOCKER_CREDENTIALS_ID = 'docker-cred'
        DOCKER_IMAGE          = 'sreedev34/maker'
        SONAR_SERVER_NAME     = 'sonarqube_env'
        SONAR_SCANNER_TOOL    = 'sonar-scanner'
        GIT_CREDENTIALS_ID    = 'github-cred'
        MANIFEST_REPO         = 'https://github.com/sreedev34/task1.git'
        MANIFEST_BRANCH       = 'main'
    }

    stages {
        stage('Checkout Source Code') {
            steps {
                //checkout scm
                echo 'passed'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool "${SONAR_SCANNER_TOOL}"

                    withSonarQubeEnv("${SONAR_SERVER_NAME}") {
                        sh "${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=aws-topology-key \
                            -Dsonar.projectName='AWS-Topology-Project' \
                            -Dsonar.sources=."
                    }
                }
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: "${DOCKER_CREDENTIALS_ID}",
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh '''
                            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

                            # Build Docker image
                            docker build \
                                -t ${DOCKER_IMAGE}:$BUILD_NUMBER \
                                -t ${DOCKER_IMAGE}:latest \
                                -f Dockerfile .

                            # Push Docker images
                            docker push ${DOCKER_IMAGE}:$BUILD_NUMBER
                            docker push ${DOCKER_IMAGE}:latest

                            docker logout
                        '''
                    }
                }
            }
        }

        stage('Update Manifest Repo') {
            steps {
                dir('manifest-workspace') {
                    git branch: "${MANIFEST_BRANCH}",
                credentialsId: "${GIT_CREDENTIALS_ID}",
                url: "${MANIFEST_REPO}"

                    sh """
                # Update aws-topology deployment image
                sed -i '/name: aws-topology/{n;s|image:.*|image: ${DOCKER_IMAGE}:${BUILD_NUMBER}|}' manifests/deployment.yaml

                git -c user.name='sreedev34' \
                    -c user.email='sreedevpattiam@gmail.com' \
                    commit -am "chore(manifest): update maker image tag to ${BUILD_NUMBER} [skip ci]"
            """

                    // Authenticate Git push using Jenkins credentials
                    withCredentials([gitUsernamePassword(
                credentialsId: "${GIT_CREDENTIALS_ID}",
                gitToolName: 'Default'
            )]) {
                        sh 'git push origin main'
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline executed successfully! Manifest repo updated with new image tag.'
        }

        failure {
            echo 'Pipeline failed. Please review the stage logs above.'
        }
    }
}
