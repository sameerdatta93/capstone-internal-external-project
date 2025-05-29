pipeline {
        agent any 
    environment {
        IMAGE_NAME = 'samdatta93/external-project-cp'
        DOCKERHUB_CREDS = 'dockerhub-creds'
        VERSION = "1.0.${env.BUILD_NUMBER}"
                AWS_REGION = 'us-east-1'
                CLUSTER_NAME = 'sd-cluster'
HOME = '/tmp'
KUBECONFIG = '/tmp/.kube/config'
LIFECYCLE = ""
CREDENTIAL_ID = ""
    }

    stages {

        stage('Determine Environment')
	{	
	parallel{
		stage('Dev Env'){
			when { 
    				expression {
					return env.GIT_BRANCH != 'main' && env.GIT_BRANCH != 'origin/main' 
				}
			}
			steps {
			 script {
				LIFECYCLE = "dev"
				CREDENTIAL_ID = "aws-creds-dev"
			 }	
			}
		}
		stage('Prod Env'){
                        when {
                                expression {
                                        return env.GIT_BRANCH == 'main' || env.GIT_BRANCH == 'origin/main'
                                }
                        }
                        steps {
                         script {
                                LIFECYCLE = "prod"
                                CREDENTIAL_ID = "aws-creds"
                         }
                        }
                }
	}
		
	}
        stage('Git Checkout') {
            steps {
                                echo 'Retrieve source from github' 
                sh """
                whoami
                pwd
                """
                checkout scm
            }
        }
stage ('Building and pushing apps')
{
parallel {
 stage ('Building external app')
{ stages{
        stage ('Install Dependencies') {
                        agent {
                docker { 
                    image 'node:14-alpine'
                    args '-e HOME=/tmp -e NPM_CONFIG_PREFIX=/tmp/.npm'
                    reuseNode true
                }
            }
            steps {
                echo 'Run npm install'
sh '''
                pwd 
		cd external
               pwd
                npm ci
'''
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
			echo "${env.GIT_BRANCH}"
			echo "${env.BRANCH_NAME}"
                    echo 'build the image' 
sh '''
cd external
pwd
'''
                    sh "docker build -f external/Dockerfile -t ${IMAGE_NAME}:${VERSION} ."
		    sh "docker tag ${IMAGE_NAME}:${VERSION} ${IMAGE_NAME}:latest"
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${DOCKERHUB_CREDS}",
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {
                    sh """
                                                echo 'push the image to docker hub' 
                        echo "${DOCKER_PASSWORD}" | docker login -u "${DOCKER_USER}" --password-stdin
                        docker push ${IMAGE_NAME}:${VERSION}
			docker push ${IMAGE_NAME}:latest
                    """
                }
            }
        }

                stage('Remove local docker image') {
            steps{
                                sh "echo 'remove local docker image'"
                sh "docker rmi ${IMAGE_NAME}:${VERSION}"
		sh "docker rmi ${IMAGE_NAME}:latest"
                //sh "docker rmi $imageName:$BUILD_NUMBER"
            }
        } }
}
stage ('Building internal app')
{
stages {
stage ('Install Dependencies') {
                        agent {
                docker { 
                    image 'node:14-alpine'
                    args '-e HOME=/tmp -e NPM_CONFIG_PREFIX=/tmp/.npm'
                    reuseNode true
                }
            }
            steps {
                                echo 'Run npm install' 
                
sh '''
                pwd 
                cd internal
               pwd
                npm ci
'''
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                                        echo 'build the image' 
sh '''
cd internal
pwd
'''
                    sh "docker build -f internal/Dockerfile -t ${IMAGE_NAME}:${VERSION} ."
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${DOCKERHUB_CREDS}",
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {
                    sh """
                                                echo 'push the image to docker hub' 
                        echo "${DOCKER_PASSWORD}" | docker login -u "${DOCKER_USER}" --password-stdin
                        docker push ${IMAGE_NAME}:${VERSION}
                    """
                }
            }
        }

                stage('Remove local docker image') {
            steps{
                                sh "echo 'remove local docker image'"
                sh "docker rmi ${IMAGE_NAME}:${VERSION}"
                //sh "docker rmi $imageName:$BUILD_NUMBER"
            }
        }
}
}
}
}
                stage ('Deploy to EKS') {
                  agent {
                docker { 
                    image 'samdatta93/aws-cli-kubectl-cp:latest'
                    args '--entrypoint="" --network host'
                }
            }
                        steps {
/*echo 'Started deploying'
                sh """
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.31.2/2024-11-15/bin/linux/amd64/kubectl
chmod +x ./kubectl
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$HOME/bin:$PATH
*/
withEnv(["NAMESPACE=${LIFECYCLE}"]){
withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: "${CREDENTIAL_ID}"]]){
echo "${LIFECYCLE}"
echo "${env.BRANCH_NAME}"
sh '''
mkdir -p $(dirname $KUBECONFIG)
aws eks update-kubeconfig --region $AWS_REGION --name $CLUSTER_NAME --kubeconfig $KUBECONFIG
pwd
kubectl get namespace $NAMESPACE || kubectl create namespace $NAMESPACE
kubectl apply -f k8s/deployment.yaml --kubeconfig=$KUBECONFIG -n $NAMESPACE
kubectl apply -f k8s/service.yaml --kubeconfig=$KUBECONFIG -n $NAMESPACE
kubectl get svc
                '''
                echo 'Deployed to EKS' 
}
}
                
            }
                }
    }

    post {
        always {
            cleanWs()
        }
    }
}
