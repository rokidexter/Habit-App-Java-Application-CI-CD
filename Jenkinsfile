
pipeline {
    agent any

    tools {
        maven 'Maven3'
    }

    environment {
        ECR_REGISTRY = '382170164329.dkr.ecr.ap-south-1.amazonaws.com'
        IMAGE_NAME   = 'habitapp'
        IMAGE_TAG    = "${env.GIT_COMMIT.take(7)}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Maven Build & Test') {
            steps {
                sh 'mvn clean verify'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        mvn sonar:sonar \
                          -Dsonar.projectKey=HabitApp \
                          -Dsonar.projectName=HabitApp
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build \
                      -t ${IMAGE_NAME}:${IMAGE_TAG} \
                      -t ${ECR_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} \
                      .
                '''
            }
        }

        stage('Push Image') {
            steps {
                sh '''
                    aws ecr get-login-password --region ap-south-1 | \
                    docker login --username AWS --password-stdin ${ECR_REGISTRY}

                    docker push ${ECR_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
                '''
            }
        }

        stage('Update Helm Configuration') {
            steps {
                sh '''
                    sed -i "s/^  tag: .*/  tag: \\"${IMAGE_TAG}\\"/" helm/habitapp/values.yaml

                    echo "Updated Helm image tag:"
                    grep -n -A3 "^image:" helm/habitapp/values.yaml
                '''
            }
        }

        stage('Commit Deployment Change') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'github-push',
                        usernameVariable: 'GIT_USERNAME',
                        passwordVariable: 'GIT_PASSWORD'
                    )
                ]) {
                    sh '''
                        git config user.name "Jenkins"
                        git config user.email "jenkins@localhost"

                        git add helm/habitapp/values.yaml

                        if git diff --cached --quiet; then
                            echo "No Helm deployment changes to commit."
                        else
                            git commit -m "Update HabitApp image to ${IMAGE_TAG}"

                            git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/rokidexter/Habit-App-Java-Application-CI-CD.git HEAD:main
                        fi
                    '''
                }
            }
        }
    }
}
