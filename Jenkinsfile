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
        KUBE_CONFIG = credentials('aks-kubeconfig')
        CENTRAL_REPO = 'https://github.com/Mazin2k2/cicd-acr-central.git'  // Central repo for Jenkinsfile and Kubernetes manifests
        APP_TO_DEPLOY = 'app1' // Default app
    }

    parameters {
        choice(name: 'APP_TO_DEPLOY', choices: ['app1', 'app2'], description: 'Choose which app to deploy')
    }

    stages {
        stage('Checkout Kubernetes Manifests') {
            steps {
                script {
                    echo "Selected app to deploy: ${params.APP_TO_DEPLOY}"
                    def k8sManifestPath = ''  // Local variable to hold Kubernetes manifest path

                    // Checkout the relevant manifest from the central repository
                    checkout scm: [
                        $class: 'GitSCM',
                        branches: [[name: '*/main']],
                        userRemoteConfigs: [[
                            url: "${CENTRAL_REPO}",
                            credentialsId: 'git_pat'
                        ]]
                    ]

                    // Set the manifest path based on the selected app
                    if (params.APP_TO_DEPLOY == 'app1') {
                        echo 'Checking out app1 Kubernetes manifests'
                        k8sManifestPath = 'app1/web-app.yaml'  // Path for app1 Kubernetes manifest
                    } else if (params.APP_TO_DEPLOY == 'app2') {
                        echo 'Checking out app2 Kubernetes manifests'
                        k8sManifestPath = 'app2/web-app.yaml'  // Path for app2 Kubernetes manifest
                    }

                    echo "Using Kubernetes manifest: ${k8sManifestPath}"

                    // Set the manifest path in the environment for later stages
                    env.K8S_MANIFEST_PATH = k8sManifestPath
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

        stage ('Deploy to AKS using kubectl') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'aks-kubeconfig', variable: 'KUBECONFIG')]) {
                        sh """
                            export KUBECONFIG=${KUBECONFIG}
                            
                            # Apply the selected Kubernetes manifest
                            kubectl apply -f ${K8S_MANIFEST_PATH}
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
