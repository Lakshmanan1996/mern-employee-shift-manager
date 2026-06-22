pipeline {

    agent any

    tools {
        maven 'Maven'
       
        
    }

    environment {

        IMAGE1       = "employee-shift-frontendapp"
        IMAGE2       = "employee-shift-backendapp"
        DOCKERHUB_USER = "lakshvar96"
        GIT_REPO = "https://github.com/Lakshmanan1996/mern-employee-shift-manager.git"
    }
    
   /* =====================================================   
   CHECKOUT
    ===================================================== */

    stages {

        stage('Checkout Code') {
            
            steps {
                checkout([$class: 'GitSCM',
                    branches: [[name: 'master']],
                    userRemoteConfigs: [[url: "${GIT_REPO}"]]
                ])
            }
        }


        stage('Stash Source') {
            
            steps {
                stash includes: '**/*', name: 'source-code'
            }
        }


        /* ===================== Build Stage ===================== */
        stage('Build') {

            steps {
                unstash 'source-code'
                
                dir('frontend') {
                    sh 'npm install'
                    sh 'npm run build'
                }
                
                dir('backend') {
                    sh 'npm install'
                }
            }
        }

        /* =====================================================
           SONARQUBE ANALYSIS
        ===================================================== */

        stage('SonarQube Analysis') {
            
            steps {
                unstash 'source-code'
                script {
                    def scannerHome = tool 'SonarQubeScanner'
                    
                    withSonarQubeEnv('sonarqube') {
                    sh """
                        ${scannerHome}/bin/sonar-scanner \
                          -Dsonar.projectKey=employee-shift-manager \
                          -Dsonar.projectName=employee-shift-manager \
                          -Dsonar.sources=backend,frontend \
                          -Dsonar.exclusions=**/node_modules/**
                        """
                    }
                }
            }
        }

        /* =====================================================
           QUALITY GATE
        ===================================================== */

        stage('Quality Gate') {
            
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        /* =====================================================
           OWASP DEPENDENCY CHECK
        ===================================================== */

        stage('OWASP Dependency Check') {
            
            steps {
                unstash 'source-code'
                dependencyCheck(
                    odcInstallation: 'OWASP-DC',
                    additionalArguments: '''
                        --scan ${WORKSPACE}
                        --format ALL
                        --out ${WORKSPACE}/dependency-check-report
                    '''
                )

                sh "ls -l dependency-check-report || true"

                dependencyCheckPublisher(
                    pattern: '**/dependency-check-report.xml'
                )
            }
        }

        /* =====================================================
           DOCKER BUILD
        ===================================================== */

        stage('Docker Build') {
            
            steps {
                unstash 'source-code'
                
                echo "Build a image for frontend-service"
                
                dir('frontend') {
                sh """
                docker build -t ${DOCKERHUB_USER}/${IMAGE1}:${BUILD_NUMBER} .
                docker tag ${DOCKERHUB_USER}/${IMAGE1}:${BUILD_NUMBER} ${DOCKERHUB_USER}/${IMAGE1}:latest 
                """
                }
                
                echo "Build a image for backend-service"

                dir('backend') {
                sh """
                docker build -t ${DOCKERHUB_USER}/${IMAGE2}:${BUILD_NUMBER} .
                docker tag ${DOCKERHUB_USER}/${IMAGE2}:${BUILD_NUMBER} ${DOCKERHUB_USER}/${IMAGE2}:latest 
                """
                }    
            }
        }

           /* =====================================================
           TRIVY IMAGE SCAN
        ===================================================== */

        stage('Trivy Scan') {
            
            steps {
                sh """
                trivy image --exit-code 0 --severity HIGH,CRITICAL ${DOCKERHUB_USER}/${IMAGE1}:${BUILD_NUMBER}
                trivy image --exit-code 0 --severity HIGH,CRITICAL ${DOCKERHUB_USER}/${IMAGE2}:${BUILD_NUMBER}
                """
            }
        }

         /* =====================================================
           DOCKER PUSH
        ===================================================== */

        stage('Push Image') {
            
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                }

                sh """
                docker push ${DOCKERHUB_USER}/${IMAGE1}:${BUILD_NUMBER}
                docker push ${DOCKERHUB_USER}/${IMAGE1}:latest
                """

                 sh """
                docker push ${DOCKERHUB_USER}/${IMAGE2}:${BUILD_NUMBER}
                docker push ${DOCKERHUB_USER}/${IMAGE2}:latest
                """
            }
        }
    }

    post {
        success {
            echo "✅ employee-shift-manager CI Pipeline SUCCESS"
        }
        failure {
            echo "❌ employee-shift-manager CI Pipeline FAILED"
        }
    }
}
