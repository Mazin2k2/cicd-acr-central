pipeline {
    agent any

    environment {
        ACR_NAME = 'testacr0909'
        ACR_URL = "${ACR_NAME}.azurecr.io"
        IMAGE_NAME = 'pyimg'
        IMAGE_TAG = "${env.BUILD_ID}"
        ACR_USERNAME = 'testacr0909'
        ACR_PASSWORD = credentials('acr-access-key')  // Jenkins secret containing your ACR password
        ACR_EMAIL = 'mazin.abdulkarimrelambda.onmicrosoft.com'  // Your ACR email
        GITHUB_REPO = 'https://github.com/Mazin2k2/cicd-acr-central.git'
        KUBE_CONFIG = credentials('aks-kubeconfig')  // Jenkins secret containing your AKS kubeconfig
        HELM_CHART_PATH = ''  // This will be set dynamically in the pipeline based on app
        APP_TO_DEPLOY = 'app1'  // Default app to deploy, can be changed dynamically
    }

    stages {
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Checkout Central Repo') {
            steps {
                checkout scm: [
                    $class: 'GitSCM',
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[
                        url: 'https://github.com/Mazin2k2/cicd-acr-central.git',
                        credentialsId: 'git_pat'  // Ensure correct credentials for GitHub access
                    ]]
                ]
            }
        }

        stage('Checkout App Code') {
            steps {
                script {
                    if (params.APP_TO_DEPLOY == 'app1') {
                        echo 'Checking out app1 repository'
                        checkout scm: [
                            $class: 'GitSCM',
                            branches: [[name: '*/main']],  // Modify as needed to match the branch in your repo
                            userRemoteConfigs: [[
                                url: 'https://github.com/Mazin2k2/cicd-acr-app1.git',
                                credentialsId: 'git_pat'  // Ensure this is the correct credentials ID in Jenkins
                            ]]
                        ]
                        // Set the correct Helm chart path for app1
                        env.HELM_CHART_PATH = 'helm/mypyapp'  // Helm chart for app1
                    } else if (params.APP_TO_DEPLOY == 'app2') {
                        echo 'Checking out app2 repository'
                        checkout scm: [
                            $class: 'GitSCM',
                            branches: [[name: '*/main']],  // Modify as needed to match the branch in your repo
                            userRemoteConfigs: [[
                                url: 'https://github.com/Mazin2k2/cicd-acr-app2.git',
                                credentialsId: 'git_pat'  // Ensure this is the correct credentials ID in Jenkins
                            ]]
                        ]
                        // Set the correct Helm chart path for app2
                        env.HELM_CHART_PATH = 'helm/mypyapp1'  // Helm chart for app2
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    // Build Docker image for app
                    echo 'Building Docker image for application...'
                    sh """
                        docker build -t ${ACR_URL}/${IMAGE_NAME}:${IMAGE_TAG} .
                    """
                }
            }
        }

        stage('Push Docker Image to ACR') {
            steps {
                script {
                    // Push Docker image to Azure Container Registry (ACR)
                    echo 'Pushing Docker image to Azure Container Registry...'
                    sh """
                        echo ${ACR_PASSWORD} | docker login ${ACR_URL} -u ${ACR_USERNAME} --password-stdin
                        docker push ${ACR_URL}/${IMAGE_NAME}:${IMAGE_TAG}
                    """
                }
            }
        }

        stage('Create Docker Registry Secret') {
            steps {
                script {
                    // Create Kubernetes Docker registry secret
                    echo 'Creating Docker registry secret in Kubernetes...'
                    sh """
                        kubectl create secret docker-registry acr-secret \
                            --docker-server=${ACR_URL} \
                            --docker-username=${ACR_USERNAME} \
                            --docker-password=${ACR_PASSWORD} \
                            --docker-email=${ACR_EMAIL}
                    """
                }
            }
        }

        stage('Deploy to AKS using Helm') {
            steps {
                script {
                    // Deploy the application to AKS using Helm
                    echo 'Deploying application to Azure Kubernetes Service (AKS) using Helm...'
                    sh """
                        helm upgrade --install ${IMAGE_NAME} ${HELM_CHART_PATH} \
                            --set image.repository=${ACR_URL}/${IMAGE_NAME} \
                            --set image.tag=${IMAGE_TAG} \
                            --kubeconfig ${KUBE_CONFIG} \
                            --namespace default
                    """
                }
            }
        }

        stage('Clean up Docker Images') {
            steps {
                script {
                    // Clean up Docker images
                    echo 'Cleaning up Docker images...'
                    sh """
                        docker rmi ${ACR_URL}/${IMAGE_NAME}:${IMAGE_TAG}
                    """
                }
            }
        }
    }

    post {
        always {
            echo 'Cleaning up Jenkins workspace...'
            cleanWs()
        }
    }
}
