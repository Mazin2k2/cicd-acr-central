pipeline {
    agent any

    environment {
        ACR_NAME = 'testacr0909'
        ACR_URL = "${ACR_NAME}.azurecr.io"
        IMAGE_NAME = 'pyimg'
        IMAGE_TAG = "${env.BUILD_ID}"
        ACR_USERNAME = 'testacr0909'
        ACR_PASSWORD = credentials('acr-access-key')
        ACR_EMAIL = 'mazin.abdulkarimrelambda.onmicrosoft.com'
        GITHUB_REPO = 'https://github.com/Mazin2k2/cicd-acr-central.git'  // Central repo for Kubernetes manifests
        KUBE_CONFIG = credentials('aks-kubeconfig')
        APP_TO_DEPLOY = 'app1' // Default app
    }

    parameters {
        choice(name: 'APP_TO_DEPLOY', choices: ['app1', 'app2'], description: 'Choose which app to deploy')
    }

    stages {
        stage('Checkout Central Repo (Kubernetes Manifests)') {
            steps {
                git branch: 'main', url: "${GITHUB_REPO}"
            }
        }

        stage('Checkout App Repo (App Code)') {
            steps {
                script {
                    echo "Selected app to deploy: ${params.APP_TO_DEPLOY}"
                    def appRepoUrl = ''
                    def appDir = ''
                    
                    // Determine which app to deploy and checkout its repository
                    if (params.APP_TO_DEPLOY == 'app1') {
                        echo 'Checking out app1 repository'
                        appRepoUrl = 'https://github.com/Mazin2k2/cicd-acr-app1.git'
                        appDir = 'app1'  // Local directory for app1
                    } else if (params.APP_TO_DEPLOY == 'app2') {
                        echo 'Checking out app2 repository'
                        appRepoUrl = 'https://github.com/Mazin2k2/cicd-acr-app2.git'
                        appDir = 'app2'  // Local directory for app2
                    }
                    
                    // Clean up existing directory before checkout
                    deleteDir()  // Clean workspace to avoid conflicts

                    // Checkout the selected app repository into the corresponding directory
                    dir(appDir) {
                        checkout scm: [
                            $class: 'GitSCM',
                            branches: [[name: '*/main']],
                            userRemoteConfigs: [[
                                url: appRepoUrl,
                                credentialsId: 'git_pat'
                            ]]
                        ]
                    }
                }
            }
        }

        stage('Login to ACR') {
            steps {
                script {
                    sh '''
                        docker login ${ACR_URL} -u ${ACR_USERNAME} -p ${ACR_PASSWORD}
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def dockerfilePath = "${WORKSPACE}/${params.APP_TO_DEPLOY}/Dockerfile"
                    echo "Building Docker image from ${dockerfilePath}"
                    sh """
                        docker build -t ${ACR_URL}/${IMAGE_NAME}:${IMAGE_TAG} -f ${dockerfilePath} ${WORKSPACE}/${params.APP_TO_DEPLOY}
                    """
                }
            }
        }

        stage('Push Docker Image to ACR') {
            steps {
                script {
                    sh """
                        docker push ${ACR_URL}/${IMAGE_NAME}:${IMAGE_TAG}
                    """
                }
            }
        }

        stage('Cleanup Existing Resources') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'aks-kubeconfig', variable: 'KUBECONFIG')]) {
                        sh '''
                            export KUBECONFIG=${KUBECONFIG}
                            
                            # Clean up the service and deployment if they exist
                            kubectl delete service python-web-app-service --ignore-not-found
                            kubectl delete deployment python-web-app --ignore-not-found
                        '''
                    }
                }
            }
        }

        stage('Create Docker Registry Secret') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'aks-kubeconfig', variable: 'KUBECONFIG')]) {
                        sh '''
                            export KUBECONFIG=${KUBECONFIG}
                            
                            # Create Docker registry secret to pull image from ACR
                            kubectl create secret docker-registry regcred \
                            --docker-server=${ACR_URL} \
                            --docker-username=${ACR_USERNAME} \
                            --docker-password=${ACR_PASSWORD} \
                            --docker-email=${ACR_EMAIL} \
                            --dry-run=client -o yaml | kubectl apply -f -
                        '''
                    }
                }
            }
        }

        stage('Deploy to AKS using kubectl') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'aks-kubeconfig', variable: 'KUBECONFIG')]) {
                        sh """
                            export KUBECONFIG=${KUBECONFIG}
                            
                            # Apply the selected Kubernetes manifest from the central repo
                            kubectl apply -f ${WORKSPACE}/app${params.APP_TO_DEPLOY == 'app1' ? 1 : 2}/web-app.yaml
                        """
                    }
                }
            }
        }

        stage('Clean up Docker Images') {
            steps {
                script {
                    sh """
                        docker rmi ${ACR_URL}/${IMAGE_NAME}:${IMAGE_TAG}
                    """
                }
            }
        }
    }

    post {
        success {
            echo 'Docker image successfully built, pushed to ACR, secret created, and deployed using Kubernetes manifests to AKS!'
        }
        failure {
            echo 'There was an error in the pipeline!'
        }
    }
}
