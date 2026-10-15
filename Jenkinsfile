pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        DOCKER_EXE = 'C:\\Users\\haris\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\docker.exe'
        KUBECTL_EXE = 'C:\\Users\\haris\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\kubectl.exe'
        POWERSHELL_EXE = 'C:\\Windows\\System32\\WindowsPowerShell\\v1.0\\powershell.exe'

        IMAGE_NAME = 'harish2064/student-placement-portal'
        DEPLOYMENT_NAME = 'placement-portal'
        CONTAINER_NAME = 'placement-portal'
        SERVICE_NAME = 'placement-portal-service'
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
                    @echo off
                    if not exist "%DOCKER_EXE%" (
                        echo ERROR: Docker executable not found.
                        exit /b 1
                    )
                    if not exist "%KUBECTL_EXE%" (
                        echo ERROR: kubectl executable not found.
                        exit /b 1
                    )
                    if not exist "%POWERSHELL_EXE%" (
                        echo ERROR: Windows PowerShell executable not found.
                        exit /b 1
                    )
                    if not exist "Dockerfile" (
                        echo ERROR: Dockerfile is missing.
                        exit /b 1
                    )
                    if not exist "index.html" (
                        echo ERROR: index.html is missing.
                        exit /b 1
                    )

                    "%DOCKER_EXE%" version
                    if errorlevel 1 exit /b 1

                    "%KUBECTL_EXE%" version --client
                    if errorlevel 1 exit /b 1

                    echo Tool checks passed.
                '''
            }
        }

        stage('Build Image') {
            steps {
                bat '''
                    @echo off
                    "%DOCKER_EXE%" build -t "%IMAGE_NAME%:%BUILD_NUMBER%" .
                    if errorlevel 1 (
                        echo ERROR: Docker image build failed.
                        exit /b 1
                    )
                    echo Docker image built successfully.
                '''
            }
        }

        stage('Smoke Test') {
            steps {
                bat '''
                    @echo off
                    set "TEST_CONTAINER=placement-smoke-%BUILD_NUMBER%"

                    "%DOCKER_EXE%" run -d --name "%TEST_CONTAINER%" -p 18081:80 "%IMAGE_NAME%:%BUILD_NUMBER%"
                    if errorlevel 1 exit /b 1

                    set "TEST_OK=0"

                    for /L %%i in (1,1,15) do (
                        "%POWERSHELL_EXE%" -NoProfile -Command "try { $r = Invoke-WebRequest -Uri 'http://localhost:18081' -UseBasicParsing -TimeoutSec 3; if ($r.StatusCode -eq 200 -and $r.Content.Contains('University Placement Portal')) { exit 0 } else { exit 1 } } catch { exit 1 }"

                        if not errorlevel 1 (
                            set "TEST_OK=1"
                            goto smoke_done
                        )

                        timeout /t 2 /nobreak >nul
                    )

                    :smoke_done
                    "%DOCKER_EXE%" rm -f "%TEST_CONTAINER%"

                    if not "%TEST_OK%"=="1" (
                        echo ERROR: Smoke test failed.
                        exit /b 1
                    )

                    echo Smoke test passed.
                '''
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub1',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    bat '''
                        @echo off
                        setlocal
                        set "DOCKER_CONFIG=%WORKSPACE%\\.docker-ci-%BUILD_NUMBER%"

                        if not exist "%DOCKER_CONFIG%" mkdir "%DOCKER_CONFIG%"

                        "%DOCKER_EXE%" login -u "%DOCKER_USER%" --password-stdin <NUL
                        rem Login is performed below using PowerShell to avoid printing the token.

                        "%POWERSHELL_EXE%" -NoProfile -NonInteractive -ExecutionPolicy Bypass -Command "$ErrorActionPreference='Stop'; $env:DOCKER_PASS | & $env:DOCKER_EXE login --username $env:DOCKER_USER --password-stdin; if ($LASTEXITCODE -ne 0) { exit 1 }"

                        if errorlevel 1 (
                            echo ERROR: Docker Hub login failed.
                            rmdir /s /q "%DOCKER_CONFIG%" 2>nul
                            exit /b 1
                        )

                        "%DOCKER_EXE%" push "%IMAGE_NAME%:%BUILD_NUMBER%"
                        if errorlevel 1 (
                            echo ERROR: Docker Hub push failed. Check repository access and token permissions.
                            rmdir /s /q "%DOCKER_CONFIG%" 2>nul
                            exit /b 1
                        )

                        rmdir /s /q "%DOCKER_CONFIG%" 2>nul
                        echo Docker image pushed successfully.
                        endlocal
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'config',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    bat '''
                        @echo off
                        setlocal
                        set "KUBECONFIG=%KUBECONFIG_FILE%"

                        if not exist "deployment.yaml" (
                            echo ERROR: deployment.yaml is missing.
                            exit /b 1
                        )

                        "%KUBECTL_EXE%" get nodes
                        if errorlevel 1 (
                            echo ERROR: Kubernetes connection failed.
                            exit /b 1
                        )

                        "%KUBECTL_EXE%" apply -f deployment.yaml
                        if errorlevel 1 exit /b 1

                        "%KUBECTL_EXE%" set image deployment/%DEPLOYMENT_NAME% %CONTAINER_NAME%=%IMAGE_NAME%:%BUILD_NUMBER%
                        if errorlevel 1 (
                            echo ERROR: Could not update the deployment image.
                            "%KUBECTL_EXE%" rollout undo deployment/%DEPLOYMENT_NAME%
                            exit /b 1
                        )

                        "%KUBECTL_EXE%" rollout status deployment/%DEPLOYMENT_NAME% --timeout=180s
                        if errorlevel 1 (
                            echo ERROR: Deployment rollout failed. Attempting rollback.
                            "%KUBECTL_EXE%" rollout undo deployment/%DEPLOYMENT_NAME%
                            exit /b 1
                        )

                        "%KUBECTL_EXE%" get deployments
                        "%KUBECTL_EXE%" get services
                        "%KUBECTL_EXE%" get pods -o wide

                        echo Kubernetes deployment completed successfully.
                        endlocal
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: Image built, tested, pushed, and deployed.'
        }

        failure {
            echo 'FAILED: Check the first error in Console Output.'
        }

        always {
            echo 'Student Placement Portal CI/CD pipeline finished.'
        }
    }
}
