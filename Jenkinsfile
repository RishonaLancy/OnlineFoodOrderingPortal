
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
                    script {
                        env.ACTIVE_COLOR = bat(
                            returnStdout: true,
                            script: '@kubectl get service %SERVICE% -o jsonpath={.spec.selector.color}'
                        ).trim().toLowerCase()

                        if (env.ACTIVE_COLOR == 'blue') {
                            env.CANDIDATE_COLOR = 'green'
                        } else if (env.ACTIVE_COLOR == 'green') {
                            env.CANDIDATE_COLOR = 'blue'
                        } else {
                            error("Unknown active color: ${env.ACTIVE_COLOR}")
                        }

                        env.CANDIDATE_DEPLOYMENT =
                            env.CANDIDATE_COLOR == 'blue'
                                ? env.BLUE_DEPLOYMENT
                                : env.GREEN_DEPLOYMENT

                        env.PREVIOUS_CANDIDATE_IMAGE = bat(
                            returnStdout: true,
                            script: "@kubectl get deployment ${env.CANDIDATE_DEPLOYMENT} -o jsonpath={.spec.template.spec.containers[0].image}"
                        ).trim()

                        if (!env.PREVIOUS_CANDIDATE_IMAGE) {
                            error('Could not determine the candidate deployment previous image.')
                        }

                        echo "Current stable color: ${env.ACTIVE_COLOR}"
                        echo "Candidate color: ${env.CANDIDATE_COLOR}"
                        echo "Candidate deployment: ${env.CANDIDATE_DEPLOYMENT}"
                        echo "Previous candidate image: ${env.PREVIOUS_CANDIDATE_IMAGE}"
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
                        bat "kubectl set image deployment/${env.CANDIDATE_DEPLOYMENT} portal=${env.IMAGE}:${env.BUILD_NUMBER}"

                        bat "kubectl rollout status deployment/${env.CANDIDATE_DEPLOYMENT} --timeout=180s"
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
                        def page = bat(
                            returnStdout: true,
                            script: "@kubectl exec deployment/${env.CANDIDATE_DEPLOYMENT} -- wget -q -O - http://127.0.0.1/"
                        )

                        if (!page.contains('Foodie Express')) {
                            error('Candidate validation failed. Stable traffic has not been switched.')
                        }

                        if (!page.contains('Margherita Pizza')) {
                            error('Candidate validation failed: Margherita Pizza was not found.')
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
                        bat "kubectl set selector service/${env.SERVICE} app=food-ordering-portal,color=${env.CANDIDATE_COLOR}"

                        echo "Traffic switched to ${env.CANDIDATE_COLOR}."

                        bat 'kubectl get service food-ordering-portal-service'
                        bat 'kubectl get deployments'
                        bat 'kubectl get pods -o wide'

                        echo 'Portal URL: http://localhost:30082'
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'CI/CD pipeline completed successfully.'
            echo 'The candidate passed validation and was promoted.'
        }

        failure {
            echo 'Pipeline failed. Starting rollback.'

            script {
                if (env.ACTIVE_COLOR &&
                    env.CANDIDATE_COLOR &&
                    env.CANDIDATE_DEPLOYMENT) {

                    withCredentials([
                        file(
                            credentialsId: 'kubernetes-config',
                            variable: 'KUBECONFIG'
                        )
                    ]) {

                        // Restore traffic to the previously stable color.
                        def trafficStatus = bat(
                            returnStatus: true,
                            script: "kubectl set selector service/${env.SERVICE} app=food-ordering-portal,color=${env.ACTIVE_COLOR}"
                        )

                        if (trafficStatus != 0) {
                            echo 'WARNING: Could not restore the stable Service selector.'
                        } else {
                            echo "Traffic restored to ${env.ACTIVE_COLOR}."
                        }

                        // Restore the candidate deployment's previous image.
                        if (env.PREVIOUS_CANDIDATE_IMAGE?.trim()) {

                            def imageStatus = bat(
                                returnStatus: true,
                                script: "kubectl set image deployment/${env.CANDIDATE_DEPLOYMENT} portal=${env.PREVIOUS_CANDIDATE_IMAGE}"
                            )

                            if (imageStatus == 0) {
                                def rolloutStatus = bat(
                                    returnStatus: true,
                                    script: "kubectl rollout status deployment/${env.CANDIDATE_DEPLOYMENT} --timeout=180s"
                                )

                                if (rolloutStatus == 0) {
                                    echo 'Candidate image rollback completed.'
                                } else {
                                    echo 'WARNING: Candidate rollback rollout did not complete.'
                                }
                            } else {
                                echo 'WARNING: Could not restore the candidate deployment image.'
                            }
                        } else {
                            echo 'No previous candidate image was saved.'
                        }
                    }
                } else {
                    echo 'Deployment state was not captured. Automatic rollback was skipped.'
                }

                echo 'Check the console output and Kubernetes status.'
            }
        }

        always {
            echo 'Food Ordering Portal pipeline execution finished.'
        }
    }
}
