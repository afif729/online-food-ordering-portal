pipeline {
    agent any

    parameters {
        booleanParam(
            name: 'SIMULATE_BAD_RELEASE',
            defaultValue: false,
            description: 'Use the deliberately faulty image to demonstrate rollback'
        )
    }

    environment {
        IMAGE_NAME = 'afif729/online-food-ordering-portal'
        IMAGE_TAG = "build-${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                bat 'docker build -t %IMAGE_NAME%:%IMAGE_TAG% .'
            }
        }

        stage('Test Docker Image') {
            steps {
                bat '''
                    @echo off
                    docker rm -f foodiehub-test 2>NUL
                    docker run -d --name foodiehub-test %IMAGE_NAME%:%IMAGE_TAG%
                    if errorlevel 1 exit /b 1

                    docker exec foodiehub-test sh -c "wget -q -O - http://127.0.0.1/healthz | grep -q healthy"
                    if errorlevel 1 (
                        docker logs foodiehub-test
                        exit /b 1
                    )

                    docker exec foodiehub-test sh -c "wget -q -O - http://127.0.0.1/ | grep -q FoodieHub"
                    if errorlevel 1 (
                        docker logs foodiehub-test
                        exit /b 1
                    )

                    echo Image tests passed.
                '''
            }
            post {
                always {
                    bat 'docker rm -f foodiehub-test 2>NUL'
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKERHUB_USER',
                        passwordVariable: 'DOCKERHUB_PASSWORD'
                    )
                ]) {
                    bat '''
                        @echo off
                        echo %DOCKERHUB_PASSWORD%| docker login -u %DOCKERHUB_USER% --password-stdin
                        if errorlevel 1 exit /b 1
                        docker push %IMAGE_NAME%:%IMAGE_TAG%
                        if errorlevel 1 exit /b 1
                        docker tag %IMAGE_NAME%:%IMAGE_TAG% %IMAGE_NAME%:latest
                        docker push %IMAGE_NAME%:latest
                    '''
                }
            }
        }

        stage('Blue-Green Deployment') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubeconfig-file',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    script {
                        withEnv(["KUBECONFIG=${env.KUBECONFIG_FILE}"]) {

                            // Bootstrap Kubernetes resources on the first run only.
                            def serviceExists = bat(
                                returnStatus: true,
                                script: '@kubectl get service food-ordering-service >NUL 2>&1'
                            )

                            if (serviceExists != 0) {
                                bat 'kubectl apply -f k8s/blue-deployment.yaml'
                                bat 'kubectl apply -f k8s/green-deployment.yaml'
                                bat 'kubectl apply -f k8s/service.yaml'
                                bat 'kubectl rollout status deployment/food-ordering-blue --timeout=180s'
                            }

                            // Find the color currently receiving traffic.
                            def active = bat(
                                returnStdout: true,
                                script: '@kubectl get service food-ordering-service -o jsonpath="{.spec.selector.color}"'
                            ).trim()

                            if (!(active in ['blue', 'green'])) {
                                error("Unexpected active color: ${active}")
                            }

                            env.ACTIVE_COLOR = active
                            env.TARGET_COLOR = active == 'blue' ? 'green' : 'blue'
                            env.TARGET_DEPLOYMENT = "food-ordering-${env.TARGET_COLOR}"

                            // Keep the stable color serving traffic.
                            // Prepare the inactive color for the new release.
                            def releaseImage = "${env.IMAGE_NAME}:${env.IMAGE_TAG}"

                            if (params.SIMULATE_BAD_RELEASE) {
                                releaseImage = "${env.IMAGE_NAME}:faulty"
                            }

                            try {
                                bat "kubectl scale deployment/${env.TARGET_DEPLOYMENT} --replicas=0"

                                bat "kubectl set image deployment/${env.TARGET_DEPLOYMENT} foodiehub=${releaseImage}"

                                bat "kubectl scale deployment/${env.TARGET_DEPLOYMENT} --replicas=3"

                                // A broken image or failed readiness probe fails here.
                                bat "kubectl rollout status deployment/${env.TARGET_DEPLOYMENT} --timeout=90s"

                                // Test the candidate pod before sending user traffic to it.
                                def pod = bat(
                                    returnStdout: true,
                                    script: "@kubectl get pods -l app=foodiehub,color=${env.TARGET_COLOR} -o jsonpath=\"{.items[0].metadata.name}\""
                                ).trim()

                                if (!pod) {
                                    error('No candidate pod was found')
                                }

                                bat "kubectl exec ${pod} -- wget -q -O - http://127.0.0.1/healthz"
                                bat "kubectl exec ${pod} -- sh -c \"wget -q -O - http://127.0.0.1/ | grep -q FoodieHub\""

                                // Switch traffic only after the candidate passes.
                                bat "kubectl set selector service/food-ordering-service app=foodiehub,color=${env.TARGET_COLOR}"

                                bat 'kubectl get endpoints food-ordering-service'
                                bat 'kubectl get deployments'
                                bat 'kubectl get pods -o wide'

                                echo "Release deployed successfully. Active color: ${env.TARGET_COLOR}"
                            } catch (err) {
                                echo "Deployment failed. Restoring traffic to ${env.ACTIVE_COLOR}."

                                // Restore traffic to the previously active color.
                                bat(
                                    returnStatus: true,
                                    script: "kubectl set selector service/food-ordering-service app=foodiehub,color=${env.ACTIVE_COLOR}"
                                )

                                // Attempt to undo the candidate deployment revision.
                                bat(
                                    returnStatus: true,
                                    script: "kubectl rollout undo deployment/${env.TARGET_DEPLOYMENT}"
                                )

                                // Keep the failed candidate out of service.
                                bat(
                                    returnStatus: true,
                                    script: "kubectl scale deployment/${env.TARGET_DEPLOYMENT} --replicas=0"
                                )

                                error("Release failed. Traffic remains on ${env.ACTIVE_COLOR}.")
                            }
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'FoodieHub CI/CD pipeline finished.'
        }
    }
}