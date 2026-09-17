
pipeline {
    agent any

    tools {
        maven 'Maven-3.9.12'
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
    }
}

