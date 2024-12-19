pipeline {
    agent any

    parameters {
        choice(name: 'APP_TO_DEPLOY', choices: ['app1', 'app2'], description: 'Select which app to deploy')
    }

    environment {
        // You can define your environment variables here (e.g., ACR_PASSWORD, KUBECONFIG) if needed
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
                    } else {
                        echo 'Checking out app2 repository'
                        git 'https://github.com/Mazin2k2/cicd-acr-app2.git'
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building Docker image for ${params.APP_TO_DEPLOY}"
                    sh "docker build -t testacr0909.azurecr.io/${params.APP_TO_DEPLOY}:latest ."
                }
            }
        }

        stage('Push Docker Image to ACR') {
            steps {
                script {
                    echo "Pushing Docker image to ACR: testacr0909.azurecr.io/${params.APP_TO_DEPLOY}:latest"
                    sh "docker push testacr0909.azurecr.io/${params.APP_TO_DEPLOY}:latest"
                }
            }
        }

        stage('Create Docker Registry Secret') {
            steps {
                script {
                    echo "Creating or updating Docker registry secret in AKS"
                    sh """
                        kubectl create secret docker-registry regcred \
                            --docker-server=testacr0909.azurecr.io \
                            --docker-username=testacr0909 \
                            --docker-password=\$ACR_PASSWORD \
                            --docker-email=mazin.abdulkarimrelambda.onmicrosoft.com \
                            --dry-run=client -o yaml | kubectl apply -f -
                    """
                }
            }
        }

        stage('Deploy to AKS using Helm') {
            steps {
                script {
                    echo "Deploying ${params.APP_TO_DEPLOY} to AKS with image: testacr0909.azurecr.io/${params.APP_TO_DEPLOY}:latest"
                    
                    // Verify the existence of the chart directory
                    if (params.APP_TO_DEPLOY == 'app1') {
                        echo "Checking if helm chart for app1 exists in helm/mypyapp"
                        sh "ls helm/mypyapp"
                    } else {
                        echo "Checking if helm chart for app2 exists in helm/mypyapp1"
                        sh "ls helm/mypyapp1"
                    }

                    // Deploy with Helm
                    if (params.APP_TO_DEPLOY == 'app1') {
                        sh """
                            helm upgrade --install app1 helm/mypyapp \
                            --set appimage=testacr0909.azurecr.io/app1:latest \
                            --set apptag=latest
                        """
                    } else {
                        sh """
                            helm upgrade --install app2 helm/mypyapp1 \
                            --set appimage=testacr0909.azurecr.io/app2:latest \
                            --set apptag=latest
                        """
                    }
                }
            }
        }

        stage('Clean up Docker Images') {
            steps {
                echo "Cleaning up Docker images"
                sh "docker rmi testacr0909.azurecr.io/${params.APP_TO_DEPLOY}:latest"
            }
        }
    }

    post {
        always {
            echo 'Cleaning up...'
        }
    }
}
