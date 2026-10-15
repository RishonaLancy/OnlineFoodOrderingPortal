pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
    }

    environment {
        PATH = "C:\\Users\\risho\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin;${env.PATH}"
        IMAGE = 'rishonalancy/food-ordering-portal'
        SERVICE = 'food-ordering-portal-service'
        BLUE_DEPLOYMENT = 'food-ordering-portal-blue'
        GREEN_DEPLOYMENT = 'food-ordering-portal-green'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/RishonaLancy/OnlineFoodOrderingPortal.git'
            }
        }

        stage('Automated Testing') {
            steps {
                powershell '''
                    $ErrorActionPreference = "Stop"

                    if (!(Test-Path "index.html")) {
                        throw "index.html is missing"
                    }

                    $html = Get-Content "index.html" -Raw

                    $required = @(
                        "Foodie Express",
                        "Margherita Pizza",
                        "South Indian Meals",
                        "FOOD20",
                        "Currently Unavailable"
                    )

                    foreach ($item in $required) {
                        if (!$html.Contains($item)) {
                            throw "Test failed: Missing $item"
                        }
                        Write-Host "PASS: Found $item"
                    }

                    if (!(Test-Path "Dockerfile")) {
                        throw "Dockerfile is missing"
                    }

                    if (!(Test-Path "deployment.yaml")) {
                        throw "deployment.yaml is missing"
                    }

                    Write-Host "All website tests passed."
                '''
            }
        }

        stage('Docker Build') {
            steps {
                bat 'docker build -t %IMAGE%:%BUILD_NUMBER% .'
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKERHUB_USER',
                        passwordVariable: 'DOCKERHUB_PASS'
                    )
                ]) {
                    bat '''
                        @echo off
                        echo %DOCKERHUB_PASS%| docker login -u %DOCKERHUB_USER% --password-stdin
                        if errorlevel 1 exit /b 1
                        docker push %IMAGE%:%BUILD_NUMBER%
                        if errorlevel 1 exit /b 1
                        docker logout
                    '''
                }
            }
        }

        stage('Prepare Kubernetes') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubernetes-config',
                        variable: 'KUBECONFIG'
                    )
                ]) {
                    bat 'kubectl apply -f deployment.yaml'

                    script {
                        env.ACTIVE_COLOR = bat(
                            returnStdout: true,
                            script: '@kubectl get service food-ordering-portal-service -o jsonpath={.spec.selector.color}'
                        ).trim().toLowerCase()

                        if (env.ACTIVE_COLOR == 'blue') {
                            env.CANDIDATE_COLOR = 'green'
                        } else if (env.ACTIVE_COLOR == 'green') {
                            env.CANDIDATE_COLOR = 'blue'
                        } else {
                            error("Unknown active color: ${env.ACTIVE_COLOR}")
                        }

                        echo "Current stable version: ${env.ACTIVE_COLOR}"
                        echo "New version will be deployed to: ${env.CANDIDATE_COLOR}"
                    }
                }
            }
        }

        stage('Deploy Candidate') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubernetes-config',
                        variable: 'KUBECONFIG'
                    )
                ]) {
                    script {
                        def deployment = env.CANDIDATE_COLOR == 'blue'
                            ? env.BLUE_DEPLOYMENT
                            : env.GREEN_DEPLOYMENT

                        bat "kubectl set image deployment/${deployment} portal=${env.IMAGE}:${env.BUILD_NUMBER}"

                        bat "kubectl rollout status deployment/${deployment} --timeout=180s"
                    }
                }
            }
        }

        stage('Validate Candidate') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubernetes-config',
                        variable: 'KUBECONFIG'
                    )
                ]) {
                    script {
                        def deployment = env.CANDIDATE_COLOR == 'blue'
                            ? env.BLUE_DEPLOYMENT
                            : env.GREEN_DEPLOYMENT

                        def page = bat(
                            returnStdout: true,
                            script: "@kubectl exec deployment/${deployment} -- wget -q -O - http://127.0.0.1/"
                        )

                        if (!page.contains('Foodie Express')) {
                            error('Candidate validation failed. Stable traffic remains on the old version.')
                        }

                        echo 'Candidate validation passed.'
                    }
                }
            }
        }

        stage('Switch Traffic') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubernetes-config',
                        variable: 'KUBECONFIG'
                    )
                ]) {
                    script {
                        bat "kubectl set selector service/${env.SERVICE} app=food-ordering-portal color=${env.CANDIDATE_COLOR}"

                        bat 'kubectl get service food-ordering-portal-service'
                        bat 'kubectl get deployments'
                        bat 'kubectl get pods -o wide'

                        echo "Traffic switched to ${env.CANDIDATE_COLOR}."
                        echo 'Portal URL: http://localhost:30082'
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'CI/CD pipeline completed successfully.'
        }
        failure {
            echo 'Pipeline failed. Check the stage logs before attempting another deployment.'
            echo 'The traffic switch occurs only after candidate validation passes.'
        }
        always {
            echo 'Food Ordering Portal pipeline execution finished.'
        }
    }
}
