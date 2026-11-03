pipeline {
    agent any

    tools {
        jdk 'jdk'
        nodejs 'node'
    }

    environment {
    SCANNER_HOME = tool 'SonarScanner'
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
                    url: 'https://github.com/gsharma1298/netflix-clone.git'
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
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=netflix \
                        -Dsonar.projectKey=netflix
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    timeout(time: 5, unit: 'MINUTES') {
                        waitForQualityGate(
                            abortPipeline: false
                        )
                    }
                }
            }
        }

        stage('OWASP FS Scan') {
            steps {
                dependencyCheck(
                    additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit --noupdate',
                    odcInstallation: 'dp-check'
                )
                dependencyCheckPublisher(
                    pattern: '**/dependency-check-report.xml'
                )
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs . > trivyfs.txt'
            }
        }
    
    
    stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker') {
                       sh "docker build --no-cache --build-arg TMDB_V3_API_KEY=82f8ec5199456f6d6b4a79e61f14a43c -t netflix:${BUILD_NUMBER} . "
                       sh "docker tag netflix:${BUILD_NUMBER} gsharma1298/netflix:${BUILD_NUMBER} "
                       sh "docker push gsharma1298/netflix:${BUILD_NUMBER} "
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image gsharma1298/netflix:${BUILD_NUMBER} > trivyimage.txt"
            }
        }
        
        stage("run app"){
            steps{
                sh '''
                docker rm -f netflix
                docker run -d --name netflix -p 8081:80 gsharma1298/netflix:${BUILD_NUMBER}
                '''
            }
        }
        
}

    post {
        always {
            emailext(
                attachLog: true,
                subject: "${currentBuild.currentResult}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """Project: ${env.JOB_NAME}
Build Number: ${env.BUILD_NUMBER}
Status: ${currentBuild.currentResult}
URL: ${env.BUILD_URL}
""",
                to: 'gsharma1298@gmail.com'
            )
        }
    }
}
