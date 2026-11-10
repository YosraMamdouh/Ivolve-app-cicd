@Library('ivolve-shared-library') _

pipeline {
    agent { label 'docker-agent' }

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
        stage('Unit Test')   { steps { script { buildApp.test() } } }
        stage('Build app')   { steps { script { buildApp.build() } } }

        stage('Build Docker Image') { steps { script { buildImage.build(env.IMAGE_NAME, env.IMAGE_TAG) } } }
        stage('Scan Docker Image')  { steps { script { buildImage.scan(env.IMAGE_NAME, env.IMAGE_TAG) } } }
        stage('Push image to DockerHub') { steps { script { buildImage.push(env.IMAGE_NAME, env.IMAGE_TAG) } } }
        stage('Delete image locally')    { steps { script { buildImage.delete(env.IMAGE_NAME, env.IMAGE_TAG) } } }

        stage('Update image in deployment file') { steps { script { deployOnK8s.updateImage(env.IMAGE_NAME, env.IMAGE_TAG) } } }
        stage('Deploy to Kubernetes') { steps { script { deployOnK8s.deploy(env.K8S_SERVER, env.K8S_NAMESPACE) } } }
    }

    post {
        always {
            echo "Pipeline finished: ${currentBuild.currentResult} (build #${BUILD_NUMBER})"
            script { buildImage.cleanup(env.IMAGE_NAME, env.IMAGE_TAG) }
        }
        success { echo "SUCCESS: ${IMAGE_NAME}:${IMAGE_TAG} was pushed and deployed to the ${K8S_NAMESPACE} namespace." }
        failure { echo "FAILURE: the pipeline failed. Check the console output of the failed stage." }
    }
}
