pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'nodejs16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        APP_NAME = "reddit-clone-pipeline"
        RELEASE = "1.0.0"
        DOCKER_USER = "kashyapvishal184"
        DOCKER_PASS = credentials('docker-token') // Use credentials function
        IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
        JENKINS_API_TOKEN = credentials("test-git")
    }
    stages {
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/18april/a-reddit-clone.git'
            }
        }
        stage("Sonarqube Analysis") {
            steps {
                withSonarQubeEnv('SonarQube-Server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=reddit-clone-ci \
                    -Dsonar.projectKey=reddit-clone-ci'''
                }
            }
        }
        stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage("Build & Push Docker Image") {
            steps {
                script {
                    docker.withRegistry('', DOCKER_PASS) {
                        docker_image = docker.build "${IMAGE_NAME}"
                        docker_image.push("${IMAGE_TAG}")
                        docker_image.push('latest')
                    }
                }
            }
        }
        stage("Trivy Image Scan") {
            steps {
                script {
                    sh '''docker run -v /var/run/docker.sock:/var/run/docker.sock \
                    aquasec/trivy image ${IMAGE_NAME}:latest --no-progress \
                    --scanners vuln --exit-code 0 --severity HIGH,CRITICAL --format table > trivyimage.txt'''
                }
            }
        }
        stage ('Cleanup Artifacts') {
            steps {
                script {
                    sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker rmi ${IMAGE_NAME}:latest"
                }
            }
        }
        stage("Trigger CD Pipeline") {
            steps {
                script {
                    sh '''curl -v -k --user clouduser:${JENKINS_API_TOKEN} -X POST \
                    -H 'cache-control: no-cache' -H 'content-type: application/x-www-form-urlencoded' \
                    --data 'IMAGE_TAG=${IMAGE_TAG}' 'http://ec2-65-2-187-142.ap-south-1.compute.amazonaws.com:8080/job/Reddit-Clone-CD/buildWithParameters?token=gitops-token' '''
                }
            }
        }
    }
    post {
        always {
            emailext attachLog: true,
                subject: "'${currentBuild.result}'",
                body: "Project: ${env.JOB_NAME}<br/>" +
                      "Build Number: ${env.BUILD_NUMBER}<br/>" +
                      "URL: ${env.BUILD_URL}<br/>",
                to: 'kashyapvishal0418@gmail.com',
                attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
}
