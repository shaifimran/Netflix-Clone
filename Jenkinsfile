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
                    sh """
                        export PATH=${NODE_HOME}/bin:\$PATH
                        npm install
                    """
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
                    // Poll SonarQube manually
                    def SONAR_TOKEN = sh(script: "echo \$SONAR_TOKEN", returnStdout: true).trim()
                    def SONAR_PROJECT_KEY = 'netflix-app'

                    // Get the task ID of the latest analysis
                    def SONAR_TASK_ID = sh(
                        script: """curl -s -u squ_f786105b44326c311e2109378b4996037fcb8d84: \
                        http://98.83.253.163:9000/api/ce/task?componentKey=${SONAR_PROJECT_KEY} | jq -r '.task.id'""",
                        returnStdout: true
                    ).trim()
                    echo "SonarQube task ID: ${SONAR_TASK_ID}"

                    // Poll until analysis is complete (timeout ~5 minutes)
                    def maxAttempts = 30
                    def attempt = 0
                    def status = ''
                    while (attempt < maxAttempts) {
                        def response = sh(
                            script: "curl -s -u squ_f786105b44326c311e2109378b4996037fcb8d84: http://98.83.253.163:9000/api/ce/task?id=${SONAR_TASK_ID}",
                            returnStdout: true
                        ).trim()
                        status = sh(script: "echo '${response}' | jq -r '.task.status'", returnStdout: true).trim()
                        echo "Attempt ${attempt+1}: SonarQube task status = ${status}"
                        if (status == 'SUCCESS' || status == 'FAILED' || status == 'CANCELED') {
                            break
                        }
                        attempt++
                        sleep 10
                    }

                    if (status != 'SUCCESS') {
                        error "SonarQube analysis did not complete successfully: ${status}"
                    }

                    // Get actual Quality Gate status
                    def analysisId = sh(script: "echo '${response}' | jq -r '.task.analysisId'", returnStdout: true).trim()
                    def qgResponse = sh(
                        script: "curl -s -u squ_f786105b44326c311e2109378b4996037fcb8d84: http://98.83.253.163:9000/api/qualitygates/project_status?analysisId=${analysisId}",
                        returnStdout: true
                    ).trim()
                    def qgStatus = sh(script: "echo '${qgResponse}' | jq -r '.projectStatus.status'", returnStdout: true).trim()
                    echo "Quality Gate status: ${qgStatus}"

                    if (qgStatus != 'OK') {
                        error "Pipeline failed due to Quality Gate status: ${qgStatus}"
                    }
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
