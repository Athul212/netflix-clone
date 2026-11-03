
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
        stage('Checkout Code') {
            steps {
                checkout scm
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
                        -Dsonar.projectKey=netflix \
                        -Dsonar.projectName=netflix \
                        -Dsonar.sources=.
                    '''
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                catchError(buildResult: 'UNSTABLE',
                           stageResult: 'UNSTABLE') {
                    dependencyCheck(
                        additionalArguments: '--scan . --format XML --noupdate',
                        odcInstallation: 'dp-check'
                    )
                    dependencyCheckPublisher(
                        pattern: '**/dependency-check-report.xml'
                    )
                }
            }
        }

        stage('Trivy File System Scan') {
            steps {
                sh '''
                    trivy fs \
                    --scanners vuln \
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
                    --scanners vuln \
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

        stage('Verify Deployment') {
            steps {
                sh '''
                    sleep 5
                    curl --fail --silent --show-error \
                    http://127.0.0.1:8081/ > /dev/null
                    echo "Netflix application is responding!"
                '''
            }
        }
    }

    post {
        always {
            archiveArtifacts(
                artifacts: 'trivyfs.txt,trivyimage.txt,**/dependency-check-report.xml',
                allowEmptyArchive: true
            )
            echo "Pipeline completed: ${currentBuild.currentResult}"
        }
    }
}
