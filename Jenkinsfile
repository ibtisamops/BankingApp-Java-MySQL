pipeline {
    agent any
    
    tools {
        maven 'maven3'
    }
    
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        IMAGE_TAG = "v${BUILD_NUMBER}"
    }


    stages {
        /*
        stage('Git Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/ibtisam-iq/BankingApp-Java-MySQL.git'
            }
        }
        */
        
        // Continuous Integration
        
        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
        
        stage('Testing') {
            steps {
                sh "mvn test"
            }
        }
        
        stage('Trivy FS Scan') {
            steps {
                sh "trivy fs --format table -o fs-report.html ."
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {                 // server name configured in jenkins
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \              
                    '''
                        /*
                        -Dsonar.projectName=IbtisamXbankapp \
                        -Dsonar.projectKey=IbtisamXbankapp \
                        -Dsonar.java.binaries=target \
                        -Dsonar.branch.name=ibtisam
                        */    
                }
            }
        }
        
        stage('Quality Gate Check') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }
        
        stage('Package') {
            steps {
                sh "mvn package"
            }
        }
        
        stage('Deploy Artifact To Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'IbtisamX-nexus', jdk: 'jdk17', maven: 'maven3', mavenSettingsConfig: '', traceability: false) {
                    sh "mvn deploy"
                }
            }
        }
        
        stage('Docker Image Build & Tag') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker build -t mibtisam/bankingapp-java-mysql:$IMAGE_TAG ."
                    }
                }
            }
        }
        
        stage('Scan Image') {
            steps {
                sh "trivy image --format table -o image-report.html mibtisam/bankingapp-java-mysql:$IMAGE_TAG"
            }
        }
        
        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker push mibtisam/bankingapp-java-mysql:$IMAGE_TAG"
                    }
                }
            }
        }
        
        stage('Update Manifest File') {
            steps {
                script {
                    cleanWs()         // Clean workspace before starting
                    withCredentials([usernamePassword(credentialsId: 'git', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                        sh '''
                            # Clone the Mega-Project-CD repository
                            # git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/mibtisam/Mega-Project-CD.git
                            
                            # Update the image tag in the manifest.yaml file
                            cd Mega-Project-CD
                            sed -i "s|mibtisam/bankingapp-java-mysql:.*|mibtisam/bankingapp-java-mysql:${IMAGE_TAG}|" manifest/manifest.yaml
                            
                            # Confirm changes
                            echo "Updated manifest file contents:"
                            cat manifest/manifest.yaml
                            
                            # Commit and push the changes
                            git config user.name "Ibtisam"
                            git config user.email "loveyou@ibtisam.com"
                            git add manifest/manifest.yaml
                            git commit -m "Update image tag to ${IMAGE_TAG}"
                            git push origin main
                        '''
                    }
                }
            }
        }
    }
}    
    // Post Actions

post {
    always {
        script {
            def jobName = env.JOB_NAME
            def buildNumber = env.BUILD_NUMBER
            def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
            def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'

            def body = """
                <html>
                <body>
                <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                <h2>${jobName} - Build ${buildNumber}</h2>
                <div style="background-color: ${bannerColor}; padding: 10px;">
                <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
                </div>
                <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                </div>
                </body>
                </html>
            """

            emailext (
                subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                body: body,
                to: 'tempmail@gmail.com',
                from: 'ibtisamx@tempmail.com',
                replyTo: 'ibtisamx@tempmail.com',
                mimeType: 'text/html',
               
            )
        }
    }
}

