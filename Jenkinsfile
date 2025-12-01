pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'nodejs16'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKERHUB_CREDENTIALS = 'docker'
        DOCKER_USERNAME = 'shaifimran'
        TMDB_API_KEY = 'a7e149dffa03bc3f6db3ada1e39c51ce'
        SONAR_TOKEN_ID = 'squ_token' // Jenkins credential ID
        SONAR_PROJECT_KEY = 'netflix-app'
        SONAR_HOST_URL = 'http://98.83.253.163:9000'
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
                sh """
                    export PATH=${tool 'nodejs16'}/bin:${tool 'jdk17'}/bin:$PATH
                    npm install
                """
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withCredentials([string(credentialsId: "${SONAR_TOKEN_ID}", variable: 'SONAR_TOKEN')]) {
                    withSonarQubeEnv('SonarQube') {
                        sh """
                            ${SCANNER_HOME}/bin/sonar-scanner \
                            -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                            -Dsonar.sources=. \
                            -Dsonar.host.url=${SONAR_HOST_URL} \
                            -Dsonar.login=$SONAR_TOKEN
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    withCredentials([string(credentialsId: "${SONAR_TOKEN_ID}", variable: 'SONAR_TOKEN')]) {

                        // Get latest analysis ID
                        def analysisId = sh(
                            script: """curl -s -u ${SONAR_TOKEN}: ${SONAR_HOST_URL}/api/project_analyses/search?project=${SONAR_PROJECT_KEY} | jq -r '.analyses[0].key'""",
                            returnStdout: true
                        ).trim()
                        echo "Latest analysis ID: ${analysisId}"

                        if (!analysisId) {
                            error "Could not find latest analysis ID for project ${SONAR_PROJECT_KEY}"
                        }

                        // Poll Quality Gate status
                        def maxAttempts = 30
                        def attempt = 0
                        def qgStatus = 'PENDING'
                        while (attempt < maxAttempts && qgStatus == 'PENDING') {
                            def qgResponse = sh(
                                script: "curl -s -u ${SONAR_TOKEN}: ${SONAR_HOST_URL}/api/qualitygates/project_status?analysisId=${analysisId}",
                                returnStdout: true
                            ).trim()
                            qgStatus = sh(script: "echo '${qgResponse}' | jq -r '.projectStatus.status'", returnStdout: true).trim()
                            echo "Attempt ${attempt+1}: Quality Gate status = ${qgStatus}"
                            if (qgStatus != 'PENDING') { break }
                            attempt++
                            sleep 10
                        }

                        if (qgStatus != 'OK') {
                            error "Pipeline failed due to Quality Gate status: ${qgStatus}"
                        }

                        echo "Quality Gate passed: ${qgStatus}"
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
                sh "trivy fs . > trivy-fs-report.txt || true"
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: "${DOCKERHUB_CREDENTIALS}") {
                        sh """
                            docker build --build-arg TMDB_V3_API_KEY=${TMDB_API_KEY} -t netflix .
                            docker tag netflix ${DOCKER_USERNAME}/netflix:latest
                            docker push ${DOCKER_USERNAME}/netflix:latest
                        """
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh "trivy image ${DOCKER_USERNAME}/netflix:latest > trivy-image-report.txt || true"
            }
        }

        stage('Deploy Container') {
            steps {
                sh """
                    docker rm -f netflix || true
                    docker run -d --name netflix -p 8081:80 ${DOCKER_USERNAME}/netflix:latest
                """
            }
        }

    }

    post {
        always {
            archiveArtifacts artifacts: '**/trivy-*-report.txt, **/dependency-check-report.xml', allowEmptyArchive: true
        }
    }
}

