pipeline {
    agent any

    environment {
        AWS_REGION = "ap-south-1"
        ECR_REPO = "494249241129.dkr.ecr.ap-south-1.amazonaws.com/demo-repo"
        IMAGE_NAME = "demo-node"
        IMAGE_TAG = "latest"

        BASTION_HOST = "13.233.227.235"
        BASTION_USER = "deployagent"

        REMOTE_APP_DIR = "/home/deployagent/node-app"
        MANIFEST = "/home/deployagent/kubernetes-manifest/demo-node.yaml"
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
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'bastion-ssh',
                        keyFileVariable: 'PEM_FILE',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {

                    sh '''
                    ssh -i $PEM_FILE -o StrictHostKeyChecking=no $SSH_USER@$BASTION_HOST "
                        mkdir -p $REMOTE_APP_DIR
                        rm -rf $REMOTE_APP_DIR/*
                    "

                    scp -i $PEM_FILE -o StrictHostKeyChecking=no -r * \
                    $SSH_USER@$BASTION_HOST:$REMOTE_APP_DIR/
                    '''
                }
            }
        }

        stage('Build Docker Image and Push to ECR') {
            steps {
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'bastion-ssh',
                        keyFileVariable: 'PEM_FILE',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {

                    sh '''
                    ssh -i $PEM_FILE -o StrictHostKeyChecking=no $SSH_USER@$BASTION_HOST "

                        echo 'Logging into ECR'
                        aws ecr get-login-password --region $AWS_REGION | \
                        docker login --username AWS --password-stdin $ECR_REPO

                        echo 'Building Docker image'
                        cd $REMOTE_APP_DIR

                        docker build -t $IMAGE_NAME:$IMAGE_TAG .

                        echo 'Tagging image'
                        docker tag $IMAGE_NAME:$IMAGE_TAG $ECR_REPO:$IMAGE_TAG

                        echo 'Pushing image'
                        docker push $ECR_REPO:$IMAGE_TAG
                    "
                    '''
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'bastion-ssh',
                        keyFileVariable: 'PEM_FILE',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {

                    sh '''
                    ssh -i $PEM_FILE -o StrictHostKeyChecking=no $SSH_USER@$BASTION_HOST "

                        echo 'Updating manifest'
                        sed -i 's|image:.*|image: $ECR_REPO:$IMAGE_TAG|' $MANIFEST

                        echo 'Deploying to EKS'
                        #kubectl apply -f $MANIFEST

                        echo 'Waiting for rollout'
                        kubectl rollout restart deployment/demo-node
                    "
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'bastion-ssh',
                        keyFileVariable: 'PEM_FILE',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {

                    sh '''
                    ssh -i $PEM_FILE -o StrictHostKeyChecking=no $SSH_USER@$BASTION_HOST "

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

