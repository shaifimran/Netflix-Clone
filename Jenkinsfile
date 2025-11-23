pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKERHUB_CREDENTIALS = 'docker'      // your credentials ID
        SONARQUBE_SERVER = 'sonar-server'     // your SonarQube name in Jenkins
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout Code') {
            steps {
                git branch: 'main', url: '<YOUR_GITHUB_REPO_URL>'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh """
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=netflix-app \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=http://<YOUR-SONAR-IP>:9000 \
                        -Dsonar.login=<YOUR-TOKEN>
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '''
                    --scan .
                    --format XML
                ''',
                outdir: 'dependency-check-report'
                
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh "trivy fs . > trivyfs-report.txt"
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: DOCKERHUB_CREDENTIALS, toolName: 'docker') {
                        sh """
                            docker build --build-arg TMDB_V3_API_KEY=<YOUR_API_KEY> -t netflix .
                            docker tag netflix <your-dockerhub-username>/netflix:latest
                            docker push <your-dockerhub-username>/netflix:latest
                        """
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh "trivy image <your-dockerhub-username>/netflix:latest > trivy-image-report.txt"
            }
        }

        stage('Deploy Container') {
            steps {
                sh """
                    docker rm -f netflix || true
                    docker run -d --name netflix -p 8081:80 <your-dockerhub-username>/netflix:latest
                """
            }
        }

    }
}
