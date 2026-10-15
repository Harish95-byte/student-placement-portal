
pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 20, unit: 'MINUTES')
    }

    environment {
        DOCKER_DIR = 'C:\\Users\\haris\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin'
        PS_EXE = 'C:\\Windows\\System32\\WindowsPowerShell\\v1.0\\powershell.exe'

        IMAGE = 'harish2064/student-placement-portal'
        DEPLOYMENT = 'placement-portal'
        CONTAINER_NAME = 'placement-portal'
        SERVICE = 'placement-portal-service'
        NODE_PORT = '30080'
        TEST_PORT = '18080'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                bat 'git --version'
            }
        }

        stage('Check Tools') {
            steps {
                bat '''
                    "%DOCKER_DIR%\\docker.exe" --version
                    kubectl version --client
                    "%PS_EXE%" -NoProfile -Command "$PSVersionTable.PSVersion.ToString()"
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                bat '''
                    "%DOCKER_DIR%\\docker.exe" build --pull=false -t %IMAGE%:%BUILD_NUMBER% .
                    if errorlevel 1 exit /b 1
                    "%DOCKER_DIR%\\docker.exe" tag %IMAGE%:%BUILD_NUMBER% %IMAGE%:latest
                '''
            }
        }

        stage('Test Docker Image') {
            steps {
                script {
                    try {
                        bat '''
                            "%DOCKER_DIR%\\docker.exe" rm -f placement-ci-test 2>nul
                            "%DOCKER_DIR%\\docker.exe" run -d --name placement-ci-test -p %TEST_PORT%:80 %IMAGE%:%BUILD_NUMBER%
                            if errorlevel 1 exit /b 1
                        '''

                        bat '''
                            "%PS_EXE%" -NoProfile -ExecutionPolicy Bypass -Command "$ok=$false; for($i=0; $i -lt 20; $i++) { try { $r=Invoke-WebRequest -Uri 'http://127.0.0.1:%TEST_PORT%/' -TimeoutSec 3; if($r.StatusCode -eq 200 -and $r.Content -match 'Placement Portal') { $ok=$true; break } } catch {}; Start-Sleep -Seconds 2 }; if(-not $ok) { Write-Error 'Smoke test failed'; exit 1 }; Write-Output 'Smoke test passed'"
                            if errorlevel 1 exit /b 1
                        '''
                    } finally {
                        bat '''
                            "%DOCKER_DIR%\\docker.exe" rm -f placement-ci-test 2>nul
                            exit /b 0
                        '''
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
                    bat '''
                        "%PS_EXE%" -NoProfile -ExecutionPolicy Bypass -Command "$env:DH_TOKEN | & '%DOCKER_DIR%\\docker.exe' login -u $env:DH_USER --password-stdin; if($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }"
                        if errorlevel 1 exit /b 1

                        "%DOCKER_DIR%\\docker.exe" push %IMAGE%:%BUILD_NUMBER%
                        if errorlevel 1 exit /b 1

                        "%DOCKER_DIR%\\docker.exe" push %IMAGE%:latest
                        if errorlevel 1 exit /b 1
                    '''
                }
            }
        }

        stage('Deploy and Verify') {
            steps {
                script {
                    withCredentials([
                        file(
                            credentialsId: 'config',
                            variable: 'KUBECONFIG_CRED'
                        )
                    ]) {
                        bat '''
                            copy /Y "%KUBECONFIG_CRED%" "%WORKSPACE%\\kubeconfig-ci" >nul
                            if errorlevel 1 exit /b 1
                        '''

                        withEnv(["KUBECONFIG=${env.WORKSPACE}\\kubeconfig-ci"]) {
                            try {
                                bat '''
                                    kubectl config current-context
                                    kubectl get nodes
                                    if errorlevel 1 exit /b 1
                                '''

                                // Save the current image for recovery.
                                def previousImage = bat(
                                    script: 'kubectl get deployment %DEPLOYMENT% -o jsonpath={.spec.template.spec.containers[0].image}',
                                    returnStdout: true
                                ).trim()

                                // Create/update the Deployment and Service.
                                bat '''
                                    kubectl apply -f deployment.yaml
                                    if errorlevel 1 exit /b 1

                                    kubectl set image deployment/%DEPLOYMENT% %CONTAINER_NAME%=%IMAGE%:%BUILD_NUMBER%
                                    if errorlevel 1 exit /b 1
                                '''

                                try {
                                    bat '''
                                        kubectl rollout status deployment/%DEPLOYMENT% --timeout=120s
                                        if errorlevel 1 exit /b 1

                                        kubectl get pods -o wide
                                        kubectl get services
                                        kubectl get endpoints %SERVICE%
                                    '''

                                    // Verify the published NodePort.
                                    bat '''
                                        "%PS_EXE%" -NoProfile -ExecutionPolicy Bypass -Command "$ok=$false; for($i=0; $i -lt 20; $i++) { try { $r=Invoke-WebRequest -Uri 'http://127.0.0.1:%NODE_PORT%/' -TimeoutSec 4; if($r.StatusCode -eq 200 -and $r.Content -match 'Placement Portal') { $ok=$true; break } } catch {}; Start-Sleep -Seconds 3 }; if(-not $ok) { Write-Error 'NodePort verification failed'; exit 1 }; Write-Output 'NodePort verification passed'"
                                        if errorlevel 1 exit /b 1

                                        kubectl get pods
                                    '''
                                } catch (verifyError) {
                                    echo 'Deployment verification failed. Attempting rollback.'

                                    if (previousImage) {
                                        bat """
                                            kubectl set image deployment/${env.DEPLOYMENT} ${env.CONTAINER_NAME}=${previousImage}
                                            if errorlevel 1 exit /b 1
                                            kubectl rollout status deployment/${env.DEPLOYMENT} --timeout=120s
                                            if errorlevel 1 exit /b 1
                                        """
                                    } else {
                                        bat """
                                            kubectl rollout undo deployment/${env.DEPLOYMENT}
                                            if errorlevel 1 exit /b 1
                                            kubectl rollout status deployment/${env.DEPLOYMENT} --timeout=120s
                                            if errorlevel 1 exit /b 1
                                        """
                                    }

                                    throw verifyError
                                }
                            } finally {
                                bat '''
                                    if exist "%WORKSPACE%\\kubeconfig-ci" del /F /Q "%WORKSPACE%\\kubeconfig-ci"
                                    exit /b 0
                                '''
                            }
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: Student Placement Portal built, tested, pushed, deployed and verified.'
            echo 'Open http://localhost:30080 to view the application.'
        }

        failure {
            echo 'FAILURE: Check the stage logs. Deployment verification failures trigger a rollback attempt.'
        }

        always {
            echo 'Student Placement Portal CI/CD pipeline finished.'
        }
    }
}
