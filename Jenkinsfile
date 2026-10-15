
pipeline {
    agent any

    environment {
        IMAGE = 'harish2064/student-placement-portal'
        DEPLOYMENT = 'placement-portal'
        CONTAINER = 'placement-portal'
        SERVICE = 'placement-portal-service'

        DOCKER_DIR = 'C:\\Users\\haris\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin'
        PS_DIR = 'C:\\Windows\\System32\\WindowsPowerShell\\v1.0'

        PATH = "C:\\Windows\\System32\\WindowsPowerShell\\v1.0;C:\\Users\\haris\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin;${env.PATH}"
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
                    "%DOCKER_DIR%\\docker.exe" --version
                    kubectl version --client
                    "%PS_DIR%\\powershell.exe" -NoProfile -Command "$PSVersionTable.PSVersion.ToString()"
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                bat '"%DOCKER_DIR%\\docker.exe" build -t %IMAGE%:%BUILD_NUMBER% .'
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
                                for ($i = 0; $i -lt 15; $i++) {
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
                                    throw "Local portal verification failed"
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
                    file(credentialsId: 'config', variable: 'KUBECONFIG')
                ]) {
                    script {
                        try {
                            bat 'kubectl apply -f deployment.yaml'
                            bat 'kubectl set image deployment/%DEPLOYMENT% %CONTAINER%=%IMAGE%:%BUILD_NUMBER%'
                            bat 'kubectl rollout status deployment/%DEPLOYMENT% --timeout=120s'

                            powershell '''
                                $pf = $null
                                $ok = $false

                                try {
                                    $pf = Start-Process kubectl `
                                        -ArgumentList @("port-forward", "service/placement-portal-service", "18082:80") `
                                        -PassThru -WindowStyle Hidden

                                    for ($i = 0; $i -lt 20; $i++) {
                                        if ($pf.HasExited) {
                                            throw "Port forwarding exited unexpectedly"
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
                                        throw "Kubernetes portal verification failed"
                                    }
                                } finally {
                                    if ($pf -and -not $pf.HasExited) {
                                        Stop-Process -Id $pf.Id -Force
                                    }
                                }
                            '''

                            bat 'kubectl get pods'
                        } catch (err) {
                            echo 'Deployment verification failed. Attempting rollback.'

                            bat '''
                                kubectl rollout undo deployment/placement-portal
                                kubectl rollout status deployment/placement-portal --timeout=120s
                                kubectl get pods
                            '''

                            error("Deployment failed; rollback attempted. ${err}")
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
