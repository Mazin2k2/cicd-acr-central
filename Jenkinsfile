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
        GITHUB_REPO = 'https://github.com/Mazin2k2/cicd-azure-jenkins.git'
        KUBE_CONFIG = credentials('aks-kubeconfig')
        APP_TO_DEPLOY = 'app1' // Default app
    }

    parameters {
        choice(name: 'APP_TO_DEPLOY', choices: ['app1', 'app2'], description: 'Choose which app to deploy')
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: "${GITHUB_REPO}"
            }
        }

        stage('Checkout App Code') {
            steps {
                script {
                    echo "Selected app to deploy: ${params.APP_TO_DEPLOY}"
                    def appYamlPath = '' // Local variable to hold the app YAML file path
                    if (params.APP_TO_DEPLOY == 'app1') {
                        echo 'Checking out app1 repository'
                        checkout scm: [
                            $class: 'GitSCM',
                            branches: [[name: '*/main']],
                            userRemoteConfigs: [[
                                url: 'https://github.com/Mazin2k2/cicd-acr-app1.git',
                                credentialsId: 'git_pat'
                            ]]
                        ]
                        appYamlPath = 'app1/web-app.yaml'  // Path for app1 YAML file
                    } else if (params.APP_TO_DEPLOY == 'app2') {
                        echo 'Checking out app2 repository'
                        checkout scm: [
                            $class: 'GitSCM',
                            branches: [[name: '*/main']],
                            userRemoteConfigs: [[
                                url: 'https://github.com/Mazin2k2/cicd-acr-app2.git',
                                credentialsId: 'git_pat'
                            ]]
                        ]
                        appYamlPath = 'app2/web-app.yaml'  // Path for app2 YAML file
                    }
                    echo "Using app YAML file: ${appYamlPath}"

                    // Set the app YAML file path in the environment for later stages
                    env.APP_YAML_PATH = appYamlPath
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

        // First Clean Up: Delete conflicting resources (service and deployment)
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

        // Second Clean Up: Create Docker Registry Secret for AKS
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

                            # Substitute the image and tag into the deployment YAML file dynamically
                            sed -i 's|{{IMAGE_NAME}}|${ACR_URL}/${IMAGE_NAME}|g' ${APP_YAML_PATH}
                            sed -i 's|{{IMAGE_TAG}}|${IMAGE_TAG}|g' ${APP_YAML_PATH}

                            # Deploy to AKS using kubectl
                            kubectl apply -f ${APP_YAML_PATH} --record
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
            echo 'Docker image successfully built, pushed to ACR, secret created, and deployed using kubectl to AKS!'
        }
        failure {
            echo 'There was an error in the pipeline!'
        }
    }
}
