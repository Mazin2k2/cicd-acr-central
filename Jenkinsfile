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
        GITHUB_REPO_APP1 = 'https://github.com/Mazin2k2/cicd-azure-jenkins-app1.git'
        GITHUB_REPO_APP2 = 'https://github.com/Mazin2k2/cicd-azure-jenkins-app2.git'
        GITHUB_REPO_MANIFESTS = 'https://github.com/Mazin2k2/cicd-acr-central.git'  // The repo containing manifests
        KUBE_CONFIG = credentials('aks-kubeconfig')  // Jenkins secret containing your AKS kubeconfig
    }

    parameters {
        choice(name: 'APP_TO_DEPLOY', choices: ['app1', 'app2'], description: 'Choose which app to deploy')
    }

    stages {
        stage('Checkout Code') {
            steps {
                script {
                    // Checkout the correct application repository based on selected app
                    if (params.APP_TO_DEPLOY == 'app1') {
                        git credentialsId: 'git_pat', branch: 'main', url: "${GITHUB_REPO_APP1}"
                    } else if (params.APP_TO_DEPLOY == 'app2') {
                        git credentialsId: 'git_pat', branch: 'main', url: "${GITHUB_REPO_APP2}"
                    }

                    // Checkout the manifests repo (cicd-acr-central.git) which contains the deployment YAML files
                    dir('manifests') {
                        git credentialsId: 'git_pat', branch: 'main', url: "${GITHUB_REPO_MANIFESTS}"
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
                    // Set IMAGE_NAME dynamically based on the selected app
                    if (params.APP_TO_DEPLOY == 'app1') {
                        IMAGE_NAME = 'pyimg-app1'
                    } else if (params.APP_TO_DEPLOY == 'app2') {
                        IMAGE_NAME = 'pyimg-app2'
                    }

                    // Build Docker image based on selected app
                    sh """
                        docker build -t ${ACR_URL}/${IMAGE_NAME}:${IMAGE_TAG} .
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

        stage('Clean up Previous Resources') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'aks-kubeconfig', variable: 'KUBECONFIG')]) {
                        sh """
                            export KUBECONFIG=${KUBECONFIG}

                            # Clean up previous deployment, service, ingress, and secret if they exist
                            kubectl delete deployment python-web-app --ignore-not-found=true
                            kubectl delete service python-web-app-service --ignore-not-found=true
                            kubectl delete ingress python-web-app-ingress --ignore-not-found=true
                            kubectl delete secret regcred --ignore-not-found=true
                        """
                    }
                }
            }
        }

        stage('Create Docker Registry Secret') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'aks-kubeconfig', variable: 'KUBECONFIG')]) {
                        sh """
                            export KUBECONFIG=${KUBECONFIG}

                            # Create or update the docker registry secret
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

        stage('Deploy to AKS') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'aks-kubeconfig', variable: 'KUBECONFIG')]) {
                        sh '''
                            export KUBECONFIG=${KUBECONFIG}

                            # Set path to the manifest based on the selected app
                            if [ "${params.APP_TO_DEPLOY}" == "app1" ]; then
                                APP_YAML_PATH="app1/web-app.yaml"
                            elif [ "${params.APP_TO_DEPLOY}" == "app2" ]; then
                                APP_YAML_PATH="app2/web-app.yaml"
                            fi

                            # Substitute the image and tag into the web-app.yaml file dynamically
                            sed -i 's|{{IMAGE_NAME}}|${ACR_URL}/${IMAGE_NAME}|g' ${APP_YAML_PATH}
                            sed -i 's|{{IMAGE_TAG}}|${IMAGE_TAG}|g' ${APP_YAML_PATH}

                            # Deploy to AKS using kubectl
                            kubectl apply -f ${APP_YAML_PATH} --record
                        '''
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
            echo 'Docker image successfully built, pushed to ACR, secret created, and deployed to AKS for the selected application!'
        }
        failure {
            echo 'There was an error in the pipeline!'
        }
    }
}
