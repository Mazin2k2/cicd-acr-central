pipeline {
    agent any

    environment {
        ACR_NAME = 'testacr0909'
        ACR_URL = "${ACR_NAME}.azurecr.io"
        ACR_USERNAME = 'testacr0909'
        ACR_PASSWORD = credentials('acr-access-key')  // Jenkins secret containing your ACR password
        ACR_EMAIL = 'mazin.abdulkarimrelambda.onmicrosoft.com'  // Your ACR email
        GITHUB_REPO_CENTRAL = 'https://github.com/Mazin2k2/cicd-acr-central.git'
        GITHUB_REPO_APP1 = 'https://github.com/Mazin2k2/cicd-acr-app1.git'
        GITHUB_REPO_APP2 = 'https://github.com/Mazin2k2/cicd-acr-app2.git'
        KUBE_CONFIG = credentials('aks-kubeconfig')  // Jenkins secret containing your AKS kubeconfig
        HELM_CHART_PATH_APP1 = 'helm/mypyapp'  // Path to Helm chart for app1
        HELM_CHART_PATH_APP2 = 'helm/mypyapp1' // Path to Helm chart for app2
    }

    parameters {
        choice(name: 'APP_SELECTION', choices: ['app1', 'app2'], description: 'Choose the app to deploy')
    }

    stages {
        stage('Checkout Central Repo') {
            steps {
                script {
                    // Checkout the central repo that contains Helm charts
                    git branch: 'main', url: "${GITHUB_REPO_CENTRAL}"
                }
            }
        }

        stage('Checkout App Code') {
            steps {
                script {
                    // Checkout the selected app repository
                    if (params.APP_SELECTION == 'app1') {
                        echo 'Checking out app1 repository'
                        git branch: 'main', url: "${GITHUB_REPO_APP1}"
                    } else {
                        echo 'Checking out app2 repository'
                        git branch: 'main', url: "${GITHUB_REPO_APP2}"
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    // Build the Docker image for the selected app
                    def imageName = "${ACR_URL}/${params.APP_SELECTION}:latest"
                    echo "Building Docker image for ${params.APP_SELECTION}"
                    sh "docker build -t ${imageName} ."
                }
            }
        }

        stage('Push Docker Image to ACR') {
            steps {
                script {
                    def imageName = "${ACR_URL}/${params.APP_SELECTION}:latest"
                    echo "Pushing Docker image to ACR: ${imageName}"
                    // Push the Docker image to ACR
                    sh "docker push ${imageName}"
                }
            }
        }

        stage('Create Docker Registry Secret') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'aks-kubeconfig', variable: 'KUBECONFIG')]) {
                        // Create or update the Docker registry secret in AKS
                        echo "Creating or updating Docker registry secret in AKS"
                        sh """
                            export KUBECONFIG=${KUBECONFIG}

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
        }

        stage('Deploy to AKS using Helm') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'aks-kubeconfig', variable: 'KUBECONFIG')]) {
                        def imageName = "${ACR_URL}/${params.APP_SELECTION}:latest"
                        echo "Deploying ${params.APP_SELECTION} to AKS with image: ${imageName}"

                        // Set the Helm chart path based on the selected app
                        def helmChartPath = params.APP_SELECTION == 'app1' ? HELM_CHART_PATH_APP1 : HELM_CHART_PATH_APP2

                        // Deploy the selected app using Helm
                        sh """
                            export KUBECONFIG=${KUBECONFIG}
                            helm upgrade --install ${params.APP_SELECTION} ${helmChartPath} \
                            --set appimage=${imageName} \
                            --set apptag=latest
                        """
                    }
                }
            }
        }

        stage('Clean up Docker Images') {
            steps {
                script {
                    // Remove the locally built Docker image
                    def imageName = "${ACR_URL}/${params.APP_SELECTION}:latest"
                    echo "Cleaning up Docker image: ${imageName}"
                    sh "docker rmi ${imageName}"
                }
            }
        }
    }

    post {
        success {
            echo 'The deployment was successful!'
        }
        failure {
            echo 'There was an error during the pipeline execution.'
        }
    }
}
