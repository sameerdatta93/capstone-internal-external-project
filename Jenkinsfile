pipeline {
    agent any 
    environment {
        EXTERNAL_IMAGE_NAME = 'samdatta93/external-project-cp'
	INTERNAL_IMAGE_NAME = 'samdatta93/internal-project-cp'
        DOCKERHUB_CREDS = 'sd-dockerhub-creds'
        VERSION = "2.0.${env.BUILD_NUMBER}"
        AWS_REGION = 'us-east-1'
        CLUSTER_NAME = ''
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
									CREDENTIAL_ID = "sd-aws-creds-dev"
									VERSION = "3.0.${env.BUILD_NUMBER}"
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
									CREDENTIAL_ID = "sd-aws-creds-prod"
									VERSION = "4.0.${env.BUILD_NUMBER}"
							}
                        }
                }
        }

        }
        stage('Git Checkout') {
            steps {
                echo 'Retrieve source from github' 
                checkout scm
            }
        }
		stage ('Building and pushing apps')
		{
		parallel {
			stage ('Building external app')
			{ 
				stages{
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
							echo 'Build the image' 
							sh "docker build -f external/Dockerfile -t ${EXTERNAL_IMAGE_NAME}:${VERSION} ."
							sh "docker tag ${EXTERNAL_IMAGE_NAME}:${VERSION} ${EXTERNAL_IMAGE_NAME}:latest"
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
								echo 'Push the image to docker hub' 
								echo "${DOCKER_PASSWORD}" | docker login -u "${DOCKER_USER}" --password-stdin
								docker push ${EXTERNAL_IMAGE_NAME}:${VERSION}
								docker push ${EXTERNAL_IMAGE_NAME}:latest
							"""
						}
					}
				}

                stage('Remove local docker image') {
					steps{
						sh "echo 'remove local docker image'"
						sh "docker rmi ${EXTERNAL_IMAGE_NAME}:${VERSION}"
						sh "docker rmi ${EXTERNAL_IMAGE_NAME}:latest"
					}
				}
				stage ('Deploy to EKS') {
					agent {
						docker { 
							image 'samdatta93/aws-cli-kubectl-cp:latest'
							args '--entrypoint=""'
						}
					}
                    steps {
						withEnv(["NAMESPACE=${LIFECYCLE}", "VERSION=${VERSION}"]){
						withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: "${CREDENTIAL_ID}"]]){
						echo "${LIFECYCLE}"
						sh '''
						CLUSTER_NAME=$(aws eks list-clusters --query "clusters[0]" --output text)
						echo "Using Cluster: $CLUSTER_NAME"
      						echo "Using Version: ${VERSION}"
						sed "s|{VERSION}|${VERSION}|g" external/k8s/deployment.yaml > external/k8s/deployment-updated.yaml
						mkdir -p $(dirname $KUBECONFIG)
						aws eks update-kubeconfig --region $AWS_REGION --name $CLUSTER_NAME --kubeconfig $KUBECONFIG
						kubectl get namespace $NAMESPACE || kubectl create namespace $NAMESPACE
						pwd
						kubectl apply -f external/k8s/deployment-updated.yaml --kubeconfig=$KUBECONFIG -n $NAMESPACE
						kubectl apply -f external/k8s/service.yaml --kubeconfig=$KUBECONFIG -n $NAMESPACE
						'''
						echo 'Deployed to EKS' 
						}
						}
                
					}
                }
				
				}
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
								echo 'Build the image' 
								sh "docker build -f internal/Dockerfile -t ${INTERNAL_IMAGE_NAME}:${VERSION} ."
								sh "docker tag ${INTERNAL_IMAGE_NAME}:${VERSION} ${INTERNAL_IMAGE_NAME}:latest"
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
									docker push ${INTERNAL_IMAGE_NAME}:${VERSION}
									docker push ${INTERNAL_IMAGE_NAME}:latest
								"""
							}
						}
					}

                stage('Remove local docker image') {
					steps{
						sh "echo 'remove local docker image'"
						sh "docker rmi ${INTERNAL_IMAGE_NAME}:${VERSION}"
						sh "docker rmi ${INTERNAL_IMAGE_NAME}:latest"
					}
				}
				stage ('Deploy to EKS') {
					agent {
						docker { 
							image 'samdatta93/aws-cli-kubectl-cp:latest'
							args '--entrypoint=""'
						}
					}
                    steps {
						withEnv(["NAMESPACE=${LIFECYCLE}", "VERSION=${VERSION}"]){
						withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: "${CREDENTIAL_ID}"]]){
						echo "${LIFECYCLE}"
						sh '''
						CLUSTER_NAME=$(aws eks list-clusters --query "clusters[0]" --output text)
                                                echo "Using Cluster: $CLUSTER_NAME"
  						sed "s|{VERSION}|${VERSION}|g" internal/k8s/deployment.yaml > internal/k8s/deployment-updated.yaml
						mkdir -p $(dirname $KUBECONFIG)
						aws eks update-kubeconfig --region $AWS_REGION --name $CLUSTER_NAME --kubeconfig $KUBECONFIG
						kubectl get namespace $NAMESPACE || kubectl create namespace $NAMESPACE
						pwd
						kubectl apply -f internal/k8s/deployment-updated.yaml --kubeconfig=$KUBECONFIG -n $NAMESPACE
						kubectl apply -f internal/k8s/service.yaml --kubeconfig=$KUBECONFIG -n $NAMESPACE
						'''
						echo 'Deployed to EKS' 
						}
						}
                
					}
                }
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
