@Library('ivolve-shared-library') _

pipeline {
    agent { label 'docker-agent' }

    environment {
        IMAGE_NAME    = 'yosramamdouh234/ivolve-app'
        IMAGE_TAG     = "${BUILD_NUMBER}"
        GIT_REPO_PATH = 'github.com/YosraMamdouh/Ivolve-app-cicd.git'
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
        // Argo CD watches the k8s/ folder of the repository and deploys the change
        stage('Push update to GitHub') { steps { script { deployOnK8s.pushToGit(env.GIT_REPO_PATH, env.IMAGE_TAG) } } }
    }

    post {
        always {
            echo "Pipeline finished: ${currentBuild.currentResult} (build #${BUILD_NUMBER})"
            script { buildImage.cleanup(env.IMAGE_NAME, env.IMAGE_TAG) }
        }
        success { echo "SUCCESS: ${IMAGE_NAME}:${IMAGE_TAG} was pushed to Docker Hub and the deployment file was pushed to GitHub. Argo CD will deploy it." }
        failure { echo "FAILURE: the pipeline failed. Check the console output of the failed stage." }
    }
}
