
pipeline {
    agent any

    environment {
        IMAGE = 'harish2064/student-placement-portal'
        DEPLOYMENT = 'placement-portal'
        CONTAINER = 'placement-portal'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                bat 'docker build -t %IMAGE%:%BUILD_NUMBER% .'
            }
        }

        stage('Test Docker Image') {
            steps {
                bat '''
                    docker rm -f placement-ci-test 2>nul
                    docker run -d --name placement-ci-test -p 18080:80 %IMAGE%:%BUILD_NUMBER%
                '''
                powershell '''
                    try {
                        $response = Invoke-WebRequest http://localhost:18080 -TimeoutSec 15
                        if ($response.Content -notmatch 'University Placement Portal|Student Placement Portal') {
                            throw "Portal smoke test failed"
                        }
                    } finally {
                        docker rm -f placement-ci-test
                    }
                '''
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub',
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_TOKEN'
                    )
                ]) {
                    powershell '''
                        $env:DH_TOKEN | docker login -u $env:DH_USER --password-stdin
                        if ($LASTEXITCODE -ne 0) {
                            throw "Docker login failed"
                        }
                    '''
                    bat 'docker push %IMAGE%:%BUILD_NUMBER%'
                }
            }
        }

        stage('Deploy and Verify') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'config',
                        variable: 'KUBECONFIG'
                    )
                ]) {
                    script {
                        try {
                            bat 'kubectl apply -f deployment.yaml'
                            bat 'kubectl set image deployment/%DEPLOYMENT% %CONTAINER%=%IMAGE%:%BUILD_NUMBER%'
                            bat 'kubectl rollout status deployment/%DEPLOYMENT% --timeout=120s'

                            powershell '''
                                $response = Invoke-WebRequest http://localhost:8082 -TimeoutSec 20
                                if ($response.Content -notmatch 'University Placement Portal|Student Placement Portal') {
                                    throw "Deployment smoke test failed"
                                }
                            '''

                            bat 'kubectl get pods'
                        } catch (err) {
                            bat 'kubectl rollout undo deployment/placement-portal'
                            bat 'kubectl rollout status deployment/placement-portal --timeout=120s'
                            bat 'kubectl get pods'
                            error("Deployment verification failed; rollback attempted. ${err}")
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Student Placement Portal CI/CD pipeline finished.'
        }
    }
}
