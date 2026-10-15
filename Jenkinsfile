pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 25, unit: 'MINUTES')
    }

    environment {
        DOCKER_EXE = 'C:\\Users\\haris\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\docker.exe'
        KUBECTL_EXE = 'C:\\Users\\haris\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\kubectl.exe'
        PS_EXE = 'C:\\Windows\\System32\\WindowsPowerShell\\v1.0\\powershell.exe'

        IMAGE = 'harish2064/student-placement-portal'
        DEPLOYMENT = 'placement-portal'
        CONTAINER_NAME = 'placement-portal'
        SERVICE = 'placement-portal-service'

        TEST_PORT = '18080'
        VERIFY_PORT = '18082'
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
                    if not exist "%DOCKER_EXE%" (
                        echo ERROR: Docker executable not found.
                        exit /b 1
                    )

                    if not exist "%KUBECTL_EXE%" (
                        echo ERROR: kubectl executable not found.
                        exit /b 1
                    )

                    if not exist "%PS_EXE%" (
                        echo ERROR: PowerShell executable not found.
                        exit /b 1
                    )

                    "%DOCKER_EXE%" --version
                    if errorlevel 1 exit /b 1

                    "%KUBECTL_EXE%" version --client
                    if errorlevel 1 exit /b 1

                    "%DOCKER_EXE%" info
                    if errorlevel 1 exit /b 1
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                bat '''
                    "%DOCKER_EXE%" build --pull=false -t %IMAGE%:%BUILD_NUMBER% .
                    if errorlevel 1 exit /b 1

                    "%DOCKER_EXE%" tag %IMAGE%:%BUILD_NUMBER% %IMAGE%:latest
                    if errorlevel 1 exit /b 1

                    echo Docker image build completed.
                '''
            }
        }

        stage('Test Docker Image') {
            steps {
                script {
                    try {
                        bat '''
                            "%DOCKER_EXE%" rm -f placement-ci-test 2>nul

                            "%DOCKER_EXE%" run -d --name placement-ci-test %IMAGE%:%BUILD_NUMBER%
                            if errorlevel 1 exit /b 1
                        '''

                        int smokeResult = bat(
                            returnStatus: true,
                            script: '''
                                "%PS_EXE%" -NoProfile -ExecutionPolicy Bypass -Command "$ok=$false; for($i=0; $i -lt 15; $i++) { $body = & $env:DOCKER_EXE exec placement-ci-test wget -q -O - http://127.0.0.1/ 2>&1; $code=$LASTEXITCODE; $content=$body -join [Environment]::NewLine; if($code -eq 0 -and $content -match 'University Placement Portal') { $ok=$true; break }; Start-Sleep -Seconds 2 }; if(-not $ok) { Write-Output 'Container status:'; & $env:DOCKER_EXE ps -a --filter name=placement-ci-test; Write-Output 'Container logs:'; & $env:DOCKER_EXE logs placement-ci-test; Write-Error 'Docker smoke test failed'; exit 1 }; Write-Output 'Docker smoke test passed.'"
                            '''
                        )

                        if (smokeResult != 0) {
                            error('Docker smoke test failed. Review container diagnostics.')
                        }
                    } finally {
                        bat(
                            returnStatus: true,
                            script: '''
                                "%DOCKER_EXE%" rm -f placement-ci-test 2>nul
                                exit /b 0
                            '''
                        )
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
                    script {
                        def dockerConfigDir =
                            "${env.WORKSPACE}\\.docker-ci-${env.BUILD_NUMBER}"

                        try {
                            withEnv(["DOCKER_CONFIG=${dockerConfigDir}"]) {
                                bat '''
                                    if not exist "%DOCKER_CONFIG%" mkdir "%DOCKER_CONFIG%"
                                    if errorlevel 1 exit /b 1

                                    echo {}>"%DOCKER_CONFIG%\\config.json"

                                    echo Logging in to Docker Hub...

                                    "%PS_EXE%" -NoProfile -ExecutionPolicy Bypass -Command "$env:DH_TOKEN | & $env:DOCKER_EXE login -u $env:DH_USER --password-stdin; if($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }"
                                    if errorlevel 1 exit /b 1

                                    "%DOCKER_EXE%" push %IMAGE%:%BUILD_NUMBER%
                                    if errorlevel 1 exit /b 1

                                    "%DOCKER_EXE%" push %IMAGE%:latest
                                    if errorlevel 1 exit /b 1

                                    echo Docker Hub push completed.
                                '''
                            }
                        } finally {
                            bat(
                                returnStatus: true,
                                script: '''
                                    if exist "%WORKSPACE%\\.docker-ci-%BUILD_NUMBER%" rmdir /S /Q "%WORKSPACE%\\.docker-ci-%BUILD_NUMBER%"
                                    exit /b 0
                                '''
                            )
                        }
                    }
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
                        def kubeconfigPath =
                            "${env.WORKSPACE}\\kubeconfig-ci"

                        try {
                            bat '''
                                copy /Y "%KUBECONFIG_CRED%" "%WORKSPACE%\\kubeconfig-ci" >nul
                                if errorlevel 1 exit /b 1
                            '''

                            withEnv(["KUBECONFIG=${kubeconfigPath}"]) {

                                echo 'Checking Kubernetes connection...'

                                bat '''
                                    "%KUBECTL_EXE%" config current-context
                                    if errorlevel 1 exit /b 1

                                    "%KUBECTL_EXE%" get nodes
                                    if errorlevel 1 exit /b 1
                                '''

                                def previousImage = bat(
                                    returnStdout: true,
                                    script: '''
                                        "%KUBECTL_EXE%" get deployment %DEPLOYMENT% -o "jsonpath={.spec.template.spec.containers[0].image}"
                                    '''
                                ).trim()

                                echo "Previous image: ${previousImage}"

                                bat '''
                                    "%KUBECTL_EXE%" apply -f deployment.yaml
                                    if errorlevel 1 exit /b 1

                                    "%KUBECTL_EXE%" set image deployment/%DEPLOYMENT% %CONTAINER_NAME%=%IMAGE%:%BUILD_NUMBER%
                                    if errorlevel 1 exit /b 1

                                    "%KUBECTL_EXE%" rollout status deployment/%DEPLOYMENT% --timeout=120s
                                    if errorlevel 1 exit /b 1

                                    "%KUBECTL_EXE%" get deployments
                                    "%KUBECTL_EXE%" get pods -o wide
                                    "%KUBECTL_EXE%" get services
                                    "%KUBECTL_EXE%" get endpoints %SERVICE%
                                '''

                                try {
                                    int verifyResult = bat(
                                        returnStatus: true,
                                        script: '''
                                            "%PS_EXE%" -NoProfile -ExecutionPolicy Bypass -Command "$ErrorActionPreference='Stop'; $port=[int]$env:VERIFY_PORT; $log=Join-Path $env:WORKSPACE 'port-forward-ci.log'; $err=Join-Path $env:WORKSPACE 'port-forward-ci-error.log'; $p=$null; try { $p=Start-Process -FilePath $env:KUBECTL_EXE -ArgumentList @('port-forward',('service/'+$env:SERVICE),($port.ToString()+':80'),'--address','127.0.0.1') -PassThru -RedirectStandardOutput $log -RedirectStandardError $err -NoNewWindow; $ok=$false; for($i=0; $i -lt 30; $i++) { if($p.HasExited) { break }; try { $r=Invoke-WebRequest -Uri ('http://127.0.0.1:'+$port+'/') -TimeoutSec 3 -UseBasicParsing; if($r.StatusCode -eq 200 -and $r.Content -match 'University Placement Portal') { $ok=$true; break } } catch {}; Start-Sleep -Seconds 2 }; if(-not $ok) { Write-Output 'Port-forward output:'; if(Test-Path $log) { Get-Content $log }; Write-Output 'Port-forward errors:'; if(Test-Path $err) { Get-Content $err }; throw 'Kubernetes service verification failed' }; Write-Output 'Kubernetes service verification passed.' } finally { if($p -and -not $p.HasExited) { Stop-Process -Id $p.Id -Force -ErrorAction SilentlyContinue } }"
                                        '''
                                    )

                                    if (verifyResult != 0) {
                                        error('Kubernetes service verification failed.')
                                    }

                                    bat '''
                                        "%KUBECTL_EXE%" get deployment %DEPLOYMENT%
                                        "%KUBECTL_EXE%" get pods
                                    '''

                                } catch (deploymentError) {
                                    echo 'Deployment verification failed. Attempting rollback.'

                                    if (previousImage) {
                                        bat """
                                            "%KUBECTL_EXE%" set image deployment/${env.DEPLOYMENT} ${env.CONTAINER_NAME}=${previousImage}
                                            if errorlevel 1 exit /b 1

                                            "%KUBECTL_EXE%" rollout status deployment/${env.DEPLOYMENT} --timeout=120s
                                            if errorlevel 1 exit /b 1
                                        """
                                    } else {
                                        bat """
                                            "%KUBECTL_EXE%" rollout undo deployment/${env.DEPLOYMENT}
                                            if errorlevel 1 exit /b 1

                                            "%KUBECTL_EXE%" rollout status deployment/${env.DEPLOYMENT} --timeout=120s
                                            if errorlevel 1 exit /b 1
                                        """
                                    }

                                    throw deploymentError
                                }
                            }
                        } finally {
                            bat(
                                returnStatus: true,
                                script: '''
                                    if exist "%WORKSPACE%\\kubeconfig-ci" del /F /Q "%WORKSPACE%\\kubeconfig-ci"
                                    if exist "%WORKSPACE%\\port-forward-ci.log" del /F /Q "%WORKSPACE%\\port-forward-ci.log"
                                    if exist "%WORKSPACE%\\port-forward-ci-error.log" del /F /Q "%WORKSPACE%\\port-forward-ci-error.log"
                                    exit /b 0
                                '''
                            )
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: Build, smoke test, Docker Hub push, Kubernetes deployment, and service verification completed.'
        }

        failure {
            echo 'FAILURE: Inspect the first failed stage in Console Output.'
        }

        always {
            echo 'Student Placement Portal CI/CD pipeline finished.'
        }
    }
}
