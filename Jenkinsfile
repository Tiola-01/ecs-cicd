pipeline {
    agent any

    environment {
        AWS_REGION      = 'us-east-1'
        AWS_ACCOUNT_ID  = '659161126002'
        ECR_REPOSITORY  = 'ecs-cicd'
        ECS_CLUSTER     = 'ecs-cicd-cluster'
        ECS_SERVICE     = 'ecs-cicd-service'
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
                    docker build -t ${ECR_REPOSITORY}:${IMAGE_TAG} .
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

        stage('Push Image to ECR') {
            steps {
                sh '''
                    docker tag ${ECR_REPOSITORY}:${IMAGE_TAG} ${IMAGE_URI}
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

                        EXECUTION_ROLE_ARN=$(aws iam get-role \
                          --role-name ecs-cicd-ecs-execution-role \
                          --region ${AWS_REGION} \
                          --query 'Role.Arn' \
                          --output text)

                        cat > container-definitions.json <<EOF
[
  {
    "name": "ecs-cicd",
    "image": "${IMAGE_URI}",
    "essential": true,
    "portMappings": [
      {
        "containerPort": 80,
        "hostPort": 80,
        "protocol": "tcp"
      }
    ],
    "healthCheck": {
      "command": [
        "CMD-SHELL",
        "wget -q -O - http://localhost/ || exit 1"
      ],
      "interval": 30,
      "timeout": 5,
      "retries": 3,
      "startPeriod": 10
    },
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/ecs-cicd",
        "awslogs-region": "${AWS_REGION}",
        "awslogs-stream-prefix": "ecs"
      }
    }
  }
]
EOF

                        TASK_REVISION=$(aws ecs register-task-definition \
                          --family ${ECS_TASK_FAMILY} \
                          --execution-role-arn ${EXECUTION_ROLE_ARN} \
                          --network-mode awsvpc \
                          --requires-compatibilities FARGATE \
                          --cpu 256 \
                          --memory 512 \
                          --container-definitions file://container-definitions.json \
                          --region ${AWS_REGION} \
                          --query 'taskDefinition.revision' \
                          --output text)

                        echo "Registered ECS task revision: ${TASK_REVISION}"

                        aws ecs update-service \
                          --cluster ${ECS_CLUSTER} \
                          --service ${ECS_SERVICE} \
                          --task-definition ${ECS_TASK_FAMILY}:${TASK_REVISION} \
                          --region ${AWS_REGION}

                        echo "Waiting for ECS service to stabilize..."

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
            sh 'rm -f container-definitions.json'
        }
    }
}
