pipeline {
    agent any
    environment {
        AWS_ACCOUNT_ID = "123456789012"
        AWS_DEFAULT_REGION = "us-east-1"
        IMAGE_REPO_NAME = "python-test-application"
        ECS_CLUSTER_NAME = 'Python_TEST_Cluster'
        ECS_SERVICE_NAME = 'python-test-service'
        TASK_DEFINITION_NAME = 'Python_Test_td'
        CONTAINER_NAME = 'Python-test-service'
        LOG_GROUP_NAME = "/ecs/python_test_application"
        IMAGE_TAG = "latest"
        REPOSITORY_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}"
        AWS_REGION = 'us-east-1'
        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com"
        SPRING_PROFILES_ACTIVE = 'test'
        ECS_TASK_EXECUTION_ROLE_ARN = "arn:aws:iam::${AWS_ACCOUNT_ID}:role/ecsTaskExecutionRole"
        ECS_TASK_ROLE_ARN = "arn:aws:iam::${AWS_ACCOUNT_ID}:role/ecsTaskRole"
        SELENIUM_IMAGE = 'selenium/standalone-chrome:115.0'
        TEST_RESULTS = 'test-results'
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
        timeout(time: 60, unit: 'MINUTES')
    }

    stages {
        stage('Clean Workspace') { steps { cleanWs() } }

        stage('Set Image Tag') {
            steps {
                script {
                    def commit = sh(returnStdout: true, script: 'git rev-parse --short HEAD || echo unknown').trim()
                    env.IMAGE_TAG = "${commit}-${env.BUILD_ID}"
                    env.IMAGE_URI = "${REPOSITORY_URI}:${IMAGE_TAG}"
                    echo "Using image tag: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('Logging into AWS ECR') {
            steps {
                script {
                    withAWS(credentials: 'aws_credentials', region: "${AWS_REGION}") {
                        sh '''
                            aws ecr get-login-password --region ${AWS_REGION} | \
                            docker login --username AWS --password-stdin ${ECR_REGISTRY}
                        '''
                    }
                }
            }
        }

        stage('Cloning Git') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/test']],
                    userRemoteConfigs: [[credentialsId: 'git_credential', url: 'https://github.com/roshanthakur999/python-test.git']]
                ])
                sh "ls -la"
            }
        }

        stage('Python - Install deps & Unit tests') {
            steps {
                script {
                    sh '''
                      set -e
                      python3 -m venv .venv
                      . .venv/bin/activate
                      pip install --upgrade pip
                      if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
                      mkdir -p ${TEST_RESULTS}
                      if [ -d tests ] || ls *test*.py 1>/dev/null 2>&1 ; then
                        pip install pytest pytest-cov
                        pytest -q --junitxml=${TEST_RESULTS}/unit-tests.xml || (echo "Unit tests failed"; exit 1)
                      else
                        echo "No tests detected, skipping pytest."
                      fi
                    '''
                }
                junit "${TEST_RESULTS}/unit-tests.xml"
            }
        }

        stage('Selenium UI tests') {
            steps {
                script {
                    // ✅ Escaped all $ inside the shell
                    sh '''
                      mkdir -p ${TEST_RESULTS}
                      docker run -d --name selenium-ci --shm-size=2g -p 4444:4444 ${SELENIUM_IMAGE}
                      timeout=30
                      while ! curl -sS http://localhost:4444/status >/dev/null 2>&1 && [ \$timeout -gt 0 ]; do
                        echo "Waiting for Selenium... \$timeout seconds left"
                        sleep 1
                        timeout=\$((timeout-1))
                      done
                    '''
                    sh '''
                      . .venv/bin/activate || true
                      pip install pytest-selenium selenium || true
                      if [ -d tests/ui -o -d tests ]; then
                        pytest tests/ui -q --driver Remote --host http://localhost:4444/wd/hub --junitxml=${TEST_RESULTS}/ui-tests.xml || true
                      else
                        echo "No UI tests found; skipping UI tests"
                      fi
                    '''
                }
                junit allowEmptyResults: true, testResults: "${TEST_RESULTS}/ui-tests.xml"
            }
            post {
                always {
                    sh 'docker rm -f selenium-ci || true'
                    archiveArtifacts artifacts: "${TEST_RESULTS}/*", allowEmptyArchive: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh "docker build -t ${IMAGE_REPO_NAME}:${IMAGE_TAG} ."
                }
            }
        }

        stage('Push Docker Image to ECR') {
            steps {
                script {
                    withAWS(credentials: 'aws_credentials', region: "${AWS_REGION}") {
                        sh '''
                          aws ecr describe-repositories --repository-names "${IMAGE_REPO_NAME}" --region ${AWS_REGION} >/dev/null 2>&1 || \
                            aws ecr create-repository --repository-name "${IMAGE_REPO_NAME}" --region ${AWS_REGION} >/dev/null
                        '''
                    }
                    sh "docker tag ${IMAGE_REPO_NAME}:${IMAGE_TAG} ${REPOSITORY_URI}:${IMAGE_TAG}"
                    sh "docker push ${REPOSITORY_URI}:${IMAGE_TAG}"
                }
            }
        }

        stage('Register Task Definition') {
            steps {
                script {
                    def taskDefinition = """
{
  "family": "${TASK_DEFINITION_NAME}",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "${ECS_TASK_EXECUTION_ROLE_ARN}",
  "taskRoleArn": "${ECS_TASK_ROLE_ARN}",
  "containerDefinitions": [
    {
      "name": "${CONTAINER_NAME}",
      "image": "${REPOSITORY_URI}:${IMAGE_TAG}",
      "essential": true,
      "portMappings": [ { "containerPort": 8353, "protocol": "tcp" } ],
      "environment": [ { "name": "SPRING_PROFILES_ACTIVE", "value": "${SPRING_PROFILES_ACTIVE}" } ],
      "logConfiguration": { "logDriver": "awslogs", "options": { "awslogs-group": "${LOG_GROUP_NAME}", "awslogs-region": "${AWS_REGION}", "awslogs-stream-prefix": "${CONTAINER_NAME}" } }
    }
  ]
}
"""
                    writeFile(file: 'taskdef.json', text: taskDefinition)
                    withAWS(credentials: 'aws_credentials', region: "${AWS_REGION}") {
                        sh 'aws ecs register-task-definition --cli-input-json file://taskdef.json > taskreg.json'
                        sh "cat taskreg.json"
                    }
                }
            }
        }

        stage('Deploy to ECS') {
            steps {
                script {
                    withAWS(credentials: 'aws_credentials', region: "${AWS_REGION}") {
                        try {
                            def newTaskDefArn = sh(script: "jq -r '.taskDefinition.taskDefinitionArn' taskreg.json", returnStdout: true).trim()
                            echo "New task definition ARN: ${newTaskDefArn}"
                            sh "aws ecs update-service --cluster ${ECS_CLUSTER_NAME} --service ${ECS_SERVICE_NAME} --task-definition ${newTaskDefArn} --force-new-deployment"
                            timeout(time: 20, unit: 'MINUTES') {
                                waitUntil {
                                    def status = sh(script: "aws ecs describe-services --cluster ${ECS_CLUSTER_NAME} --services ${ECS_SERVICE_NAME} --query 'services[0].deployments[?status==\\\"PRIMARY\\\"].rolloutState' --output text", returnStdout: true).trim()
                                    echo "Current rolloutState: ${status}"
                                    return (status == 'COMPLETED')
                                }
                            }
                        } catch (Exception e) {
                            echo "Deployment failed: ${e.getMessage()}"
                            error "Deployment failed. Check ECS console and task logs."
                        }
                    }
                }
            }
        }

        stage('Cleanup Docker Images') {
            steps {
                script {
                    sh "docker rmi ${REPOSITORY_URI}:${IMAGE_TAG} || true"
                    sh "docker rmi ${IMAGE_REPO_NAME}:${IMAGE_TAG} || true"
                }
            }
        }
    }

    post {
        always { cleanWs() }
        success { echo "Deployment completed successfully!" }
        failure { echo "Deployment failed. Please check the logs for details." }
    }
}
