pipeline {
    agent any

    parameters {
        choice(name: 'ENVIRONMENT', choices: ['Production'], description: 'Production = branch: main')
        string(name: 'APP_VERSION', defaultValue: 'V0.0', description: 'Enter the version (e.g., V0.0)')
    }

    environment {
        DOCKER_IMAGE_PREFIX = 'dockerhubrepo-name/imagename'
        DOCKERHUB_CREDENTIALS_USR = credentials('dockerhub-username-hub')
        DOCKERHUB_CREDENTIALS_PSW = credentials('dockerhub-password-hub')
        AWS_REGION = 'us-west-2'  // Change this to your AWS region
        AUTO_SCALING_GROUP = 'CMS-AutoScaling-GRP'
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    def branchName = 'main'
                    git branch: branchName, credentialsId: 'github_password', url: 'https://github.com/git link '
                }
            }
        }

        stage('Build') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE_PREFIX}:${params.APP_VERSION} -f /home/ubuntu/websites/Dockerfile ."
            }
        }

        stage('Test Docker Image') {
            steps {
                script {
                    try {
                        sh """
                        docker run -d --rm --name Test_container -p 1000:3000 ${env.DOCKER_IMAGE_PREFIX}:${params.APP_VERSION}
                        sleep 10
                        STATUS=\$(curl -s -o /dev/null -w "%{http_code}" http://localhost:1000 || echo "000")
                        docker stop container_name || true
                        if [ "\$STATUS" -ne 200 ]; then
                            echo "Health check failed with status \$STATUS"
                            exit 1
                        else
                            echo "Health check passed with status \$STATUS"
                        fi
                        """
                    } catch (Exception e) {
                        error("Docker health check failed: ${e.getMessage()}")
                    }
                }
            }
        }


        stage('Push to Docker Hub') {
            steps {
                withCredentials([string(credentialsId: 'dockerhub-username-hub', variable: 'DOCKERHUB_CREDENTIALS_USR'),
                                 string(credentialsId: 'dockerhub-password-hub', variable: 'DOCKERHUB_CREDENTIALS_PSW')]) {
                    sh """
                    echo "$DOCKERHUB_CREDENTIALS_PSW" | docker login -u "$DOCKERHUB_CREDENTIALS_USR" --password-stdin
                    docker push ${DOCKER_IMAGE_PREFIX}:${params.APP_VERSION}
                    docker tag  ${DOCKER_IMAGE_PREFIX}:${params.APP_VERSION} ${DOCKER_IMAGE_PREFIX}:latest
                    docker push ${DOCKER_IMAGE_PREFIX}:latest
                    """
                }
                script{
                    currentBuild.description = "${DOCKER_IMAGE_PREFIX}:${params.APP_VERSION}"
                }
            }
        }



        stage('Deploy') {
            steps {
                script {
                    def dockerImageName = "${env.DOCKER_IMAGE_PREFIX}:latest"
                    def composeFilePath = '/home/ubuntu/websites/docker-compose.yml'
                    withCredentials([string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                                    string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')]) {
                        sh """
                        export AWS_ACCESS_KEY_ID=\$AWS_ACCESS_KEY_ID
                        export AWS_SECRET_ACCESS_KEY=\$AWS_SECRET_ACCESS_KEY

                        # Get only running instances in the ASG
                        INSTANCE_IDS=\$(aws ec2 describe-instances \
                            --region \$AWS_REGION \
                            --filters "Name=tag:aws:autoscaling:groupName,Values=\$AUTO_SCALING_GROUP" \
                                    "Name=instance-state-name,Values=running" \
                            --query "Reservations[*].Instances[*].InstanceId" \
                            --output text)

                        echo "Running Instance IDs: \$INSTANCE_IDS"

                        if [ -z "\$INSTANCE_IDS" ]; then
                            echo "No running instances found in Auto Scaling Group: \$AUTO_SCALING_GROUP"
                            exit 0
                        fi

                        INSTANCE_IDS_FORMATTED=\$(echo \$INSTANCE_IDS | tr ' ' ',')

                        aws ssm send-command \
                            --document-name "AWS-RunShellScript" \
                            --targets "Key=instanceIds,Values=\${INSTANCE_IDS_FORMATTED}" \
                            --parameters 'commands=["docker pull ${dockerImageName}", "docker compose -f ${composeFilePath} up -d"]' \
                            --region \$AWS_REGION
                        """
                    }
                }
            }
        }
    }
}
