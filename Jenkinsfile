pipeline {
    agent any

    tools {
        jdk 'jdk17'
    }

    environment {
        DOCKERHUB_CREDENTIALS = 'docker'
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout Code') {
            steps {
                git branch: 'master', url: 'https://github.com/shaifimran/Netflix-Clone.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                script {
                    def NODE_HOME = tool 'nodejs16'
                    sh "${NODE_HOME}/bin/npm install"
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def SCANNER_HOME = tool 'sonar-scanner'
                    withCredentials([string(credentialsId: 'squ_token', variable: 'SONAR_TOKEN')]) {
                        withSonarQubeEnv('SonarQube') {
                            sh """
                                ${SCANNER_HOME}/bin/sonar-scanner \
                                -Dsonar.projectKey=netflix-app \
                                -Dsonar.sources=. \
                                -Dsonar.host.url=http://98.83.253.163:9000 \
                                -Dsonar.login=$SONAR_TOKEN
                            """
                        }
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    def qg = waitForQualityGate abortPipeline: true
                    echo "Quality Gate status: ${qg.status}"
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--scan . --format XML', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Trivy FS Scan') {
            steps {
                script {
                    if (sh(script: "command -v trivy", returnStatus: true) == 0) {
                        sh "trivy fs . > trivy-fs-report.txt"
                    } else {
                        echo "Trivy not installed, skipping Trivy FS scan"
                    }
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    if (sh(script: "command -v docker", returnStatus: true) == 0) {
                        withDockerRegistry(credentialsId: 'docker') {
                            sh """
                                docker build -t netflix .
                                docker tag netflix shaifimran/netflix:latest
                                docker push shaifimran/netflix:latest
                            """
                        }
                    } else {
                        echo "Docker not installed, skipping Docker build & push"
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                script {
                    if (sh(script: "command -v trivy", returnStatus: true) == 0) {
                        sh "trivy image shaifimran/netflix:latest > trivy-image-report.txt"
                    } else {
                        echo "Trivy not installed, skipping Trivy image scan"
                    }
                }
            }
        }

        stage('Deploy Container') {
            steps {
                script {
                    if (sh(script: "command -v docker", returnStatus: true) == 0) {
                        sh """
                            docker rm -f netflix || true
                            docker run -d --name netflix -p 8081:80 shaifimran/netflix:latest
                        """
                    } else {
                        echo "Docker not installed, skipping deploy"
                    }
                }
            }
        }

    }
}
