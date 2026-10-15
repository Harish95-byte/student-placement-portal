pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        DOCKER_EXE = 'C:\\Users\\haris\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\docker.exe'
        KUBECTL_EXE = 'C:\\Users\\haris\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\kubectl.exe'
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
                powershell '''
                    $ErrorActionPreference = 'Stop'

                    if (!(Test-Path $env:DOCKER_EXE)) {
                        throw "Docker executable not found: $env:DOCKER_EXE"
                    }

                    if (!(Test-Path $env:KUBECTL_EXE)) {
                        throw "kubectl executable not found: $env:KUBECTL_EXE"
                    }

                    if (!(Test-Path "Dockerfile")) {
                        throw "Dockerfile is missing from the workspace."
                    }

                    if (!(Test-Path "index.html")) {
                        throw "index.html is missing from the workspace."
                    }

                    & $env:DOCKER_EXE version
                    if ($LASTEXITCODE -ne 0) {
                        throw "Docker is not available."
                    }

                    & $env:KUBECTL_EXE version --client
                    if ($LASTEXITCODE -ne 0) {
                        throw "kubectl client check failed."
                    }

                    Write-Host "Tool checks completed."
                '''
            }
        }

        stage('Build Image') {
            steps {
                powershell '''
                    $ErrorActionPreference = 'Stop'
                    $image = "$env:IMAGE_NAME`:$env:BUILD_NUMBER"

                    Write-Host "Building image: $image"

                    & $env:DOCKER_EXE build -t $image .
                    if ($LASTEXITCODE -ne 0) {
                        throw "Docker image build failed."
                    }
                '''
            }
        }

        stage('Smoke Test') {
            steps {
                powershell '''
                    $ErrorActionPreference = 'Stop'
                    $image = "$env:IMAGE_NAME`:$env:BUILD_NUMBER"
                    $container = "placement-smoke-$env:BUILD_NUMBER"
                    $testPassed = $false

                    try {
                        & $env:DOCKER_EXE run -d --name $container -p 18081:80 $image
                        if ($LASTEXITCODE -ne 0) {
                            throw "Could not start the smoke-test container."
                        }

                        for ($i = 0; $i -lt 15; $i++) {
                            Start-Sleep -Seconds 2

                            try {
                                $response = Invoke-WebRequest `
                                    -Uri "http://localhost:18081" `
                                    -UseBasicParsing `
                                    -TimeoutSec 3

                                if ($response.StatusCode -eq 200 -and
                                    $response.Content -match 'University Placement Portal') {
                                    $testPassed = $true
                                    break
                                }
                            }
                            catch {
                                # The container may need more time to start.
                            }
                        }

                        if (-not $testPassed) {
                            throw "Smoke test failed. Expected the University Placement Portal page."
                        }

                        Write-Host "Smoke test passed."
                    }
                    finally {
                        & $env:DOCKER_EXE rm -f $container 2>$null
                    }
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
                    powershell '''
                        $ErrorActionPreference = 'Stop'
                        $image = "$env:IMAGE_NAME`:$env:BUILD_NUMBER"
                        $dockerConfig = Join-Path $env:WORKSPACE ".docker-ci-$env:BUILD_NUMBER"

                        try {
                            New-Item -ItemType Directory -Path $dockerConfig -Force | Out-Null
                            $env:DOCKER_CONFIG = $dockerConfig

                            $env:DOCKER_PASS |
                                & $env:DOCKER_EXE login `
                                    --username $env:DOCKER_USER `
                                    --password-stdin

                            if ($LASTEXITCODE -ne 0) {
                                throw "Docker Hub login failed."
                            }

                            & $env:DOCKER_EXE push $image
                            if ($LASTEXITCODE -ne 0) {
                                throw "Docker Hub push failed. Check repository access and token permissions."
                            }

                            Write-Host "Successfully pushed $image"
                        }
                        finally {
                            Remove-Item Env:DOCKER_CONFIG -ErrorAction SilentlyContinue
                            Remove-Item -Path $dockerConfig -Recurse -Force -ErrorAction SilentlyContinue
                        }
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
                    powershell '''
                        $ErrorActionPreference = 'Stop'
                        $env:KUBECONFIG = $env:KUBECONFIG_FILE

                        $image = "$env:IMAGE_NAME`:$env:BUILD_NUMBER"
                        $deployment = $env:DEPLOYMENT_NAME
                        $container = $env:CONTAINER_NAME
                        $kubectl = $env:KUBECTL_EXE
                        $previousImage = ""

                        try {
                            Write-Host "Checking Kubernetes connection..."

                            & $kubectl get nodes
                            if ($LASTEXITCODE -ne 0) {
                                throw "Cannot connect to Kubernetes. Check the config credential and Docker Desktop."
                            }

                            $existingDeployment = & $kubectl get deployment $deployment --ignore-not-found -o name
                            if ($LASTEXITCODE -ne 0) {
                                throw "Could not check the existing deployment."
                            }

                            if ($existingDeployment) {
                                $previousImage = & $kubectl get deployment $deployment -o "jsonpath={.spec.template.spec.containers[0].image}"
                                if ($LASTEXITCODE -ne 0) {
                                    throw "Could not read the previous deployment image."
                                }
                            }

                            if (!(Test-Path "deployment.yaml")) {
                                throw "deployment.yaml is missing from the workspace."
                            }

                            & $kubectl apply -f deployment.yaml
                            if ($LASTEXITCODE -ne 0) {
                                throw "Applying deployment.yaml failed."
                            }

                            & $kubectl set image "deployment/$deployment" "$container=$image"
                            if ($LASTEXITCODE -ne 0) {
                                throw "Updating the Kubernetes image failed."
                            }

                            & $kubectl rollout status "deployment/$deployment" --timeout=180s
                            if ($LASTEXITCODE -ne 0) {
                                throw "Kubernetes rollout failed."
                            }

                            Write-Host "Deployment succeeded."
                            & $kubectl get deployments
                            & $kubectl get services
                            & $kubectl get pods -o wide
                        }
                        catch {
                            $failure = $_.Exception.Message
                            Write-Host "Deployment error: $failure"
                            Write-Host "Attempting rollback..."

                            if ($previousImage) {
                                & $kubectl set image "deployment/$deployment" "$container=$previousImage"
                                if ($LASTEXITCODE -eq 0) {
                                    & $kubectl rollout status "deployment/$deployment" --timeout=180s
                                }
                            }
                            else {
                                & $kubectl delete deployment $deployment --ignore-not-found
                            }

                            throw $failure
                        }
                        finally {
                            Remove-Item Env:KUBECONFIG -ErrorAction SilentlyContinue
                        }
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: Placement Portal image pushed and Kubernetes deployment completed.'
        }

        failure {
            echo 'FAILED: Check the stage error in Console Output. The pipeline stops when a required stage fails.'
        }

        always {
            echo 'Student Placement Portal CI/CD pipeline finished.'
        }
    }
}