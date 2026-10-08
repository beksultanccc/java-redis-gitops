pipeline {
    agent any

    environment {
        DOCKERHUB_USER = 'your-dockerhub'
        APP_NAME       = 'java-app'
        IMAGE_TAG      = "${BUILD_NUMBER}"
        SONAR_HOST     = 'http://sonarqube:9000'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Code Quality (SonarQube)') {
            steps {
                withSonarQubeEnv('SonarQubeServer') {
                    sh 'mvn clean verify sonar:sonar'
                }
                timeout(time: 5, unit: 'MINUTES') {
                    script {
                        waitForQualityGate abortPipeline: true
                    }
                }
            }
        }

        stage('Build & Package') {
            steps {
                sh 'mvn package -DskipTests'
                sh "docker build -t ${DOCKERHUB_USER}/${APP_NAME}:${IMAGE_TAG} ."
            }
        }

        stage('Security Scan (Trivy)') {
            steps {
                // CRITICAL немесе HIGH табылса, exit-code 1 арқылы пайплайнды тоқтатады
                sh """
                    trivy image --exit-code 1 \
                                --severity CRITICAL,HIGH \
                                --no-progress \
                                ${DOCKERHUB_USER}/${APP_NAME}:${IMAGE_TAG}
                """
            }
        }

        stage('Push to Registry') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh "echo ${PASS} | docker login -u ${USER} --password-stdin"
                    sh "docker push ${DOCKERHUB_USER}/${APP_NAME}:${IMAGE_TAG}"
                }
            }
        }

        stage('Update GitOps Repo') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'github-token', usernameVariable: 'GH_USER', passwordVariable: 'GH_TOKEN')]) {
                    sh """
                        git config user.email "ci-bot@local"
                        git config user.name "CI Bot"
                        sed -i 's|image: .*/java-app:.*|image: ${DOCKERHUB_USER}/${APP_NAME}:${IMAGE_TAG}|g' k8s/java-deployment.yaml
                        git add k8s/java-deployment.yaml
                        git commit -m "chore(ci): update java-app image to ${IMAGE_TAG} [skip ci]" || exit 0
                        git push https://${GH_TOKEN}@github.com/${GH_USER}/java-redis-gitops.git main
                    """
                }
            }
        }
    }
}