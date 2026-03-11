pipeline {
    agent any

    environment {
        REPOSITORY_URI = "${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.AWS_DEFAULT_REGION}.amazonaws.com"
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    options {
        timeout(time: 60, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
    }

    stages {

        stage('Check Commit Message') {
            steps {
                script {
                    def commitMsg = sh(
                        script: 'git log -1 --pretty=%B HEAD || echo "initial"',
                        returnStdout: true
                    ).trim()
                    echo "Commit message: ${commitMsg}"
                    if (commitMsg.contains('[skip ci]')) {
                        currentBuild.result = 'NOT_BUILT'
                        error("Skipping CI — commit contains [skip ci]")
                    }
                }
            }
        }

        stage('Checkout Code') {
            steps {
                git branch: 'main',
                    credentialsId: 'github-token',
                    url: 'https://github.com/suryamani752/mern-ecommerce.git'
                echo "Build Number: ${IMAGE_TAG}"
            }
        }

        stage('Build Images') {
            parallel {

                stage('Backend Image') {
                    steps {
                        sh """
                            docker build -f backend/Dockerfile \
                                -t ${REPOSITORY_URI}/${env.ECR_REPO_NAME_BACKEND}:${IMAGE_TAG} .
                        """
                        sh """
                            docker tag \
                                ${REPOSITORY_URI}/${env.ECR_REPO_NAME_BACKEND}:${IMAGE_TAG} \
                                ${REPOSITORY_URI}/${env.ECR_REPO_NAME_BACKEND}:latest
                        """
                    }
                }

                stage('Frontend Image') {
                    steps {
                        dir('frontend') {
                            sh """
                                docker build \
                                    -t ${REPOSITORY_URI}/${env.ECR_REPO_NAME_FRONTEND}:${IMAGE_TAG} .
                            """
                            sh """
                                docker tag \
                                    ${REPOSITORY_URI}/${env.ECR_REPO_NAME_FRONTEND}:${IMAGE_TAG} \
                                    ${REPOSITORY_URI}/${env.ECR_REPO_NAME_FRONTEND}:latest
                            """
                        }
                    }
                }

            }
        }

        stage('Push to ECR') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'aws-ecr-creds',
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )]) {
                        sh """
                            aws ecr get-login-password --region ${env.AWS_DEFAULT_REGION} \
                                | docker login --username AWS \
                                --password-stdin ${REPOSITORY_URI}
                        """
                        sh "docker push ${REPOSITORY_URI}/${env.ECR_REPO_NAME_BACKEND}:${IMAGE_TAG}"
                        sh "docker push ${REPOSITORY_URI}/${env.ECR_REPO_NAME_BACKEND}:latest"
                        sh "docker push ${REPOSITORY_URI}/${env.ECR_REPO_NAME_FRONTEND}:${IMAGE_TAG}"
                        sh "docker push ${REPOSITORY_URI}/${env.ECR_REPO_NAME_FRONTEND}:latest"
                    }
                }
            }
        }

        stage('Update Kubernetes Manifests') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'github-token',
                        usernameVariable: 'GIT_USER',
                        passwordVariable: 'GIT_TOKEN'
                    )]) {
                        sh """
                            sed -i 's|${REPOSITORY_URI}/${env.ECR_REPO_NAME_BACKEND}:.*|${REPOSITORY_URI}/${env.ECR_REPO_NAME_BACKEND}:${IMAGE_TAG}|g' k8s/3-backend.yaml
                        """
                        sh """
                            sed -i 's|${REPOSITORY_URI}/${env.ECR_REPO_NAME_FRONTEND}:.*|${REPOSITORY_URI}/${env.ECR_REPO_NAME_FRONTEND}:${IMAGE_TAG}|g' k8s/4-frontend.yaml
                        """
                        sh "git add k8s/3-backend.yaml k8s/4-frontend.yaml"
                        sh """
                            git diff --quiet --cached || git commit -m "CI: Update image tags to build ${IMAGE_TAG} [skip ci]"
                        """
                        sh """
                            git push https://${GIT_USER}:${GIT_TOKEN}@github.com/suryamani752/mern-ecommerce.git HEAD:main
                        """
                    }
                }
            }
        }

    }

    post {
        always {
            sh "docker rmi ${REPOSITORY_URI}/${env.ECR_REPO_NAME_BACKEND}:${IMAGE_TAG} || true"
            sh "docker rmi ${REPOSITORY_URI}/${env.ECR_REPO_NAME_BACKEND}:latest || true"
            sh "docker rmi ${REPOSITORY_URI}/${env.ECR_REPO_NAME_FRONTEND}:${IMAGE_TAG} || true"
            sh "docker rmi ${REPOSITORY_URI}/${env.ECR_REPO_NAME_FRONTEND}:latest || true"
            cleanWs()
        }
        success {
            echo "CI SUCCESS — Build ${IMAGE_TAG} deployed via GitOps"
        }
        failure {
            echo "CI FAILED — Check stage logs above"
        }
    }
}
