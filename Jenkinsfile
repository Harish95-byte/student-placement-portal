
pipeline {
    agent any

    environment {
        IMAGE = 'harish2064/student-placement-portal'
        DEPLOYMENT = 'placement-portal'
        CONTAINER = 'placement-portal'
        SERVICE = 'placement-portal-service'
        DOCKER_DIR = 'C:\\Users\\haris\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin'
        PATH = "${DOCKER_DIR};${env.PATH}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Check Tools') {
            steps {
                bat '''
                    echo Checking Docker CLI...
                    "%DOCKER_DIR%\\docker.exe" --version
                    echo Checking Kubernetes CLI...
                    kubectl version --client
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                bat '''
                    "%DOCKER_DIR%\\docker.exe" build -t %IMAGE%:%BUILD_NUMBER% .
                '''
            }
        }

        stage('Test Docker Image') {
            steps {
                script {
                    try {
                        bat '''
                            "%DOCKER_DIR%\\docker.exe" rm -f placement-ci-test 2>nul
                            "%DOCKER_DIR%\\docker.exe" run -d --name placement-ci-test -p 18080:80 %IMAGE%:%BUILD_NUMBER%
                        '''

                        powershell '''
                            $ok = $false
                            try {
                                for ($i = 0; $i -lt 12; $i++) {
                                    try {
                                        $r = Invoke-WebRequest "http://localhost:18080" -TimeoutSec 3
                                        if ($r.Content -match 'University Placement Portal|Student Placement Portal') {
                                            $ok = $true
                                            break
                                        }
                                    } catch {
                                        Start-Sleep -Seconds 2
                                    }
                                }

                                if (-not $ok) {
                                    throw "Local Docker smoke test failed"
                                }
                            } finally {
                                & "$env:DOCKER_DIR\\docker.exe" rm -f placement-ci-test
                            }
                        '''
                    } catch (err) {
                        bat '"%DOCKER_DIR%\\docker.exe" rm -f placement-ci-test'
                        throw err
                    }
                }
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
                        $env:DH_TOKEN | & "$env:DOCKER_DIR\\docker.exe" login -u $env:DH_USER --password-stdin
                        if ($LASTEXITCODE -ne 0) {
                            throw "Docker Hub login failed"
                        }
                    '''

                    bat '"%DOCKER_DIR%\\docker.exe" push %IMAGE%:%BUILD_NUMBER%'
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

                            bat '''
                                kubectl set image deployment/%DEPLOYMENT% %CONTAINER%=%IMAGE%:%BUILD_NUMBER%
                            '''

                            bat '''
                                kubectl rollout status deployment/%DEPLOYMENT% --timeout=120s
                            '''

                            powershell '''
                                $pf = Start-Process kubectl `
                                    -ArgumentList @("port-forward", "service/placement-portal-service", "18082:80") `
                                    -PassThru -WindowStyle Hidden

                                try {
                                    $ok = $false
                                    for ($i = 0; $i -lt 20; $i++) {
                                        if ($pf.HasExited) {
                                            throw "kubectl port-forward stopped unexpectedly"
                                        }

                                        try {
                                            $r = Invoke-WebRequest "http://localhost:18082" -TimeoutSec 3
                                            if ($r.Content -match 'University Placement Portal|Student Placement Portal') {
                                                $ok = $true
                                                break
                                            }
                                        } catch {
                                            Start-Sleep -Seconds 2
                                        }
                                    }

                                    if (-not $ok) {
                                        throw "Kubernetes application smoke test failed"
                                    }
                                } finally {
                                    if ($pf -and -not $pf.HasExited) {
                                        Stop-Process -Id $pf.Id -Force
                                    }
                                }
                            '''

                            bat 'kubectl get pods'
                        } catch (err) {
                            echo "Deployment or verification failed. Attempting rollback."

                            bat '''
                                kubectl rollout undo deployment/placement-portal
                                kubectl rollout status deployment/placement-portal --timeout=120s
                                kubectl get pods
                            '''

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
