pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
        AWS_ACCOUNT_ID = '659161126002'
        ECR_REPOSITORY = 'ecs-cicd'
        ECS_CLUSTER = 'ecs-cicd-cluster'
        ECS_SERVICE = 'ecs-cicd-service'
        ECS_TASK_FAMILY = 'ecs-cicd'
        IMAGE_TAG = "build-${BUILD_NUMBER}"
        IMAGE_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}:build-${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build \
                      -t ${ECR_REPOSITORY}:${IMAGE_TAG} \
                      "/mnt/c/Users/skid0/Project 2/cicd-wordpress"
                '''
            }
        }

        stage('Login to ECR') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-ecr'
                ]]) {
                    sh '''
                        aws ecr get-login-password \
                          --region ${AWS_REGION} | \
                        docker login \
                          --username AWS \
                          --password-stdin \
                          ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                    '''
                }
            }
        }

        stage('Push Image') {
            steps {
                sh '''
                    docker tag \
                      ${ECR_REPOSITORY}:${IMAGE_TAG} \
                      ${IMAGE_URI}

                    docker push ${IMAGE_URI}
                '''
            }
        }

        stage('Deploy to ECS') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-ecr'
                ]]) {
                    sh '''
                        echo "Deploying ${IMAGE_URI}"

                        aws ecs describe-task-definition \
                          --task-definition ${ECS_TASK_FAMILY} \
                          --region ${AWS_REGION} \
                          --query taskDefinition > task-definition.json

                        python3 - <<'PY'
import json

with open("task-definition.json") as f:
    task = json.load(f)

for container in task["containerDefinitions"]:
    if container["name"] == "ecs-cicd":
        container["image"] = "${IMAGE_URI}"

for key in [
    "taskDefinitionArn",
    "revision",
    "status",
    "requiresAttributes",
    "compatibilities"
]:
    task.pop(key, None)

with open("new-task-definition.json", "w") as f:
    json.dump(task, f)
PY

                        TASK_REVISION=$(aws ecs register-task-definition \
                          --cli-input-json file://new-task-definition.json \
                          --region ${AWS_REGION} \
                          --query 'taskDefinition.revision' \
                          --output text)

                        echo "Registered ECS task revision: ${TASK_REVISION}"

                        aws ecs update-service \
                          --cluster ${ECS_CLUSTER} \
                          --service ${ECS_SERVICE} \
                          --task-definition ${ECS_TASK_FAMILY}:${TASK_REVISION} \
                          --region ${AWS_REGION}

                        aws ecs wait services-stable \
                          --cluster ${ECS_CLUSTER} \
                          --services ${ECS_SERVICE} \
                          --region ${AWS_REGION}

                        echo "ECS deployment completed successfully."
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'rm -f task-definition.json new-task-definition.json'
        }
    }
}
