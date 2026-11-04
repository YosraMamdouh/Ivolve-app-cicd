pipeline {
    agent any

    environment {
        IMAGE_NAME    = 'yosramamdouh234/ivolve-app'
        IMAGE_TAG     = "${BUILD_NUMBER}"
        K8S_SERVER    = 'https://172.30.194.27:6443'
        K8S_NAMESPACE = 'ivolve'
    }

    options {
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
    }

    stages {
        // The source code is checked out by Jenkins before the first stage
        // (Pipeline script from SCM).

        stage('Build app') {
            steps {
                // Jenkins has no Node.js, so the build runs in a temporary Node container.
                // The jenkins_home volume is mounted at the same path so the workspace is visible.
                sh '''
                    docker run --rm -u 1000:1000 \
                        -v jenkins_home:/var/jenkins_home \
                        -w "$WORKSPACE" \
                        -e npm_config_cache=/tmp/.npm \
                        node:18-alpine \
                        sh -c "npm install --omit=dev && node --check server.js"
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $IMAGE_NAME:$IMAGE_TAG -t $IMAGE_NAME:latest .'
            }
        }

        stage('Scan Docker Image') {
            steps {
                // Reports HIGH and CRITICAL vulnerabilities. --exit-code 0 keeps the pipeline going;
                // change it to 1 to make the pipeline fail when vulnerabilities are found.
                sh '''
                    docker run --rm \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        -v trivy-cache:/root/.cache/ \
                        aquasec/trivy:latest image \
                        --severity HIGH,CRITICAL --exit-code 0 --no-progress \
                        $IMAGE_NAME:$IMAGE_TAG
                '''
            }
        }

        stage('Push image to DockerHub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                                                  usernameVariable: 'DH_USER',
                                                  passwordVariable: 'DH_PASS')]) {
                    sh '''
                        echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
                        docker push $IMAGE_NAME:$IMAGE_TAG
                        docker push $IMAGE_NAME:latest
                    '''
                }
            }
        }

        stage('Delete image locally') {
            steps {
                sh 'docker rmi $IMAGE_NAME:$IMAGE_TAG $IMAGE_NAME:latest'
            }
        }

        stage('Update image in deployment file') {
            steps {
                // Only the line of the application image changes (the MySQL init container is not touched)
                sh '''
                    sed -i "s#image: ${IMAGE_NAME}:.*#image: ${IMAGE_NAME}:${IMAGE_TAG}#" k8s/deployment.yaml
                    grep -n "image:" k8s/deployment.yaml
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                // jenkins-sa (Lab 19) may only create and update Deployments, so "replace" is used.
                withCredentials([string(credentialsId: 'k8s-jenkins-token', variable: 'K8S_TOKEN')]) {
                    sh '''
                        kubectl --kubeconfig=/dev/null --server="$K8S_SERVER" \
                            --insecure-skip-tls-verify=true --token="$K8S_TOKEN" \
                            -n "$K8S_NAMESPACE" replace -f k8s/deployment.yaml
                    '''
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline finished: ${currentBuild.currentResult} (build #${BUILD_NUMBER})"
            // Make sure no image and no login stay on the Jenkins machine, even if a stage failed
            sh '''
                docker rmi $IMAGE_NAME:$IMAGE_TAG $IMAGE_NAME:latest || true
                docker logout || true
            '''
        }
        success {
            echo "SUCCESS: $IMAGE_NAME:$IMAGE_TAG was pushed and deployed to the $K8S_NAMESPACE namespace."
        }
        failure {
            echo "FAILURE: the pipeline failed. Check the console output of the failed stage."
        }
    }
}
