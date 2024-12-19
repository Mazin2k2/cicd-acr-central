pipeline {
    agent any

    parameters {
        choice(name: 'APP_TO_DEPLOY', choices: ['app1', 'app2'], description: 'Select which app to deploy')
    }

    environment {
        // Define environment variables
        ACR_NAME = 'testacr0909'
        ACR_URL = "${ACR_NAME}.azurecr.io"
        IMAGE_NAME = 'pyimg'
        IMAGE_TAG = "${env.BUILD_ID}"
        ACR_USERNAME = 'testacr0909'
        ACR_PASSWORD = credentials('acr-access-key')  // Jenkins secret containing your ACR password
        ACR_EMAIL = 'mazin.abdulkarimrelambda.onmicrosoft.com'  // Your ACR email
        GITHUB_REPO = 'https://github.com/Mazin2k2/cicd-azure-jenkins.git'
        KUBE_CONFIG = credentials('aks-kubeconfig')  // Jenkins secret containing your AKS kubeconfig
        HELM_CHART_PATH = 'helm/mypyapp'  // Default Helm chart path for app1
    }

    stages {
        stage('Checkout Central Repo') {
            steps {
                checkout scm
            }
        }

        stage('Checkout App Code') {
            steps {
                script {
                    if (params.APP_TO_DEPLOY == 'app1') {
                        echo 'Checking out app1 repository'
                        git 'https://github.com/Mazin2k2/cicd-acr-app1.git'
                        // Set the correct Helm chart path for app1
                        env.HELM_CHART_PATH = 'helm/mypyapp'
                    } else {
                        echo 'Checking out app2 repository'
                        git 'https://github.com/Mazin2k2/cicd-acr-app2.git'
                        // Set the correct Helm chart path for app2
                        env.HELM_CHART_PATH = 'helm/mypyapp1'
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building Docker image for ${params.APP_TO_DEPLOY} with tag: ${IMAGE_TAG}"
                    sh "docker build -t ${ACR_URL}/${IMAGE_NAME}:${IMAGE_TAG} ."
                }
            }
        }

        stage('Push Docker Image to ACR') {
            steps {
                script {
                    echo "Pushing Docker image to ACR: ${ACR_URL}/${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker push ${ACR_URL}/${IMAGE_NAME}:${IMAGE_TAG}"
                }
            }
        }

        stage('Create Docker Registry Secret') {
            steps {
                script {
                    echo "Creating or updating Docker registry secret in AKS"
                    sh """
                        kubectl create secret docker-registry regcred \
                            --docker-server=${ACR_URL} \
                            --docker-username=${ACR_USERNAME} \
                            --docker-password=${ACR_PASSWORD} \
                            --docker-email=${ACR_EMAIL} \
                            --dry-run=client -o yaml | kubectl apply -f -
                    """
                }
            }
        }

        stage('Deploy to AKS using Helm') {
            steps {
                script {
                    echo "Deploying ${params.APP_TO_DEPLOY} to AKS with image: ${ACR_URL}/${IMAGE_NAME}:${IMAGE_TAG}"

                    // Verify the existence of the chart directory
                    echo "Checking if Helm chart for ${params.APP_TO_DEPLOY} exists at ${HELM_CHART_PATH}"
                    sh "ls ${HELM_CHART_PATH}"

                    // Deploy with Helm
                    if (params.APP_TO_DEPLOY == 'app1') {
                        sh """
                            helm upgrade --install app1 ${HELM_CHART_PATH} \
                            --set appimage=${ACR_URL}/${IMAGE_NAME}:${IMAGE_TAG} \
                            --set apptag=${IMAGE_TAG}
                        """
                    } else {
                        sh """
                            helm upgrade --install app2 ${HELM_CHART_PATH} \
                            --set appimage=${ACR_URL}/${IMAGE_NAME}:${IMAGE_TAG} \
                            --set apptag=${IMAGE_TAG}
                        """
                    }
                }
            }
        }

        stage('Clean up Docker Images') {
            steps {
                echo "Cleaning up Docker images"
                sh "docker rmi ${ACR_URL}/${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }
    }

    post {
        always {
            echo 'Cleaning up...'
        }
    }
}
