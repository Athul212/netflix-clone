
pipeline {
    agent any

    tools {
        jdk 'jdk'
        nodejs 'node'
    }

    environment {
        SCANNER_HOME = tool 'SonarScanner'
        DOCKER_IMAGE = 'athul212/netflix'
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/athul212/netflix-clone.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        "$SCANNER_HOME/bin/sonar-scanner" \
                        -Dsonar.projectName=netflix \
                        -Dsonar.projectKey=netflix \
                        -Dsonar.sources=src
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck(
                    additionalArguments: '--scan ./ --format XML',
                    odcInstallation: 'dp-check'
                )

                dependencyCheckPublisher(
                    pattern: '**/dependency-check-report.xml'
                )
            }
        }

        stage('Trivy File System Scan') {
            steps {
                sh '''
                    trivy fs \
                    --scanners vuln,secret \
                    --format table \
                    --output trivyfs.txt \
                    .
                '''
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(
                        credentialsId: 'docker',
                        url: 'https://index.docker.io/v1/'
                    ) {
                        sh """
                            docker build \
                            -t ${DOCKER_IMAGE}:${BUILD_NUMBER} .

                            docker push \
                            ${DOCKER_IMAGE}:${BUILD_NUMBER}
                        """
                    }
                }
            }
        }

        stage('Trivy Docker Image Scan') {
            steps {
                sh """
                    trivy image \
                    --format table \
                    --output trivyimage.txt \
                    ${DOCKER_IMAGE}:${BUILD_NUMBER}
                """
            }
        }

        stage('Deploy Netflix Application') {
            steps {
                sh """
                    docker rm -f netflix || true

                    docker run -d \
                    --name netflix \
                    --restart unless-stopped \
                    -p 127.0.0.1:8081:80 \
                    ${DOCKER_IMAGE}:${BUILD_NUMBER}
                """
            }
        }
    }

    post {
        always {
            archiveArtifacts(
                artifacts: 'trivyfs.txt,trivyimage.txt,**/dependency-check-report.xml',
                allowEmptyArchive: true
            )

            emailext(
                attachLog: true,
                subject: "${currentBuild.currentResult}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """Netflix DevSecOps Pipeline

Project: ${env.JOB_NAME}
Build Number: ${env.BUILD_NUMBER}
Status: ${currentBuild.currentResult}
Build URL: ${env.BUILD_URL}
""",
                to: 'athulpb212@gmail.com'
            )
        }
    }
}
