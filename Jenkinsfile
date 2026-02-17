pipeline {
    agent any

    environment {
        AWS_REGION = "ap-south-1"
        ECR_REPO = "494249241129.dkr.ecr.ap-south-1.amazonaws.com/demo-repo"
        IMAGE_NAME = "demo-node"
        IMAGE_TAG = "${BUILD_NUMBER}"

        BASTION = "deployagent@13.233.227.235"
        REMOTE_APP_DIR = "/home/dev/node-app"
        MANIFEST = "/home/dev/kubernetes-manifest/demo-node.yaml"
    }

    stages {

        stage('Clone Source Code') {
            steps {
                git branch: 'main',
                    credentialsId: 'github-ssh',
                    url: 'git@github.com:Aniruddha-137/node-app.git'
            }
        }

        stage('Copy Code to Bastion') {
            steps {
                sshagent(['bastion-ssh']) {

                    sh '''
                    ssh -o StrictHostKeyChecking=no ${BASTION} "
                        mkdir -p ${REMOTE_APP_DIR}
                        rm -rf ${REMOTE_APP_DIR}/*
                    "

                    scp -o StrictHostKeyChecking=no -r * ${BASTION}:${REMOTE_APP_DIR}/
                    '''
                }
            }
        }

        stage('Build Docker Image and Push to ECR') {
            steps {
                sshagent(['bastion-ssh']) {

                    sh '''
                    ssh -o StrictHostKeyChecking=no ${BASTION} "

                        echo 'Logging into ECR'
                        aws ecr get-login-password --region ${AWS_REGION} | \
                        docker login --username AWS --password-stdin ${ECR_REPO}

                        echo 'Building Docker image'
                        cd ${REMOTE_APP_DIR}

                        docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .

                        echo 'Tagging image'
                        docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${ECR_REPO}:${IMAGE_TAG}

                        echo 'Pushing image to ECR'
                        docker push ${ECR_REPO}:${IMAGE_TAG}

                    "
                    '''
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                sshagent(['bastion-ssh']) {

                    sh '''
                    ssh -o StrictHostKeyChecking=no ${BASTION} "

                        echo 'Updating Kubernetes manifest'

                        sed -i 's|image:.*|image: ${ECR_REPO}:${IMAGE_TAG}|' ${MANIFEST}

                        echo 'Deploying to EKS'

                        kubectl apply -f ${MANIFEST}

                        echo 'Waiting for rollout'

                        kubectl rollout status deployment/demo-node

                    "
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                sshagent(['bastion-ssh']) {

                    sh '''
                    ssh ${BASTION} "

                        kubectl get pods -l app=demo-node
                        kubectl get deployment demo-node

                    "
                    '''
                }
            }
        }

    }

    post {

        success {
            echo "Deployment Successful"
        }

        failure {
            echo "Deployment Failed"
        }

    }
}

