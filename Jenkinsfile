pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        SONAR_SCANNER = tool 'sonar-scanner'
        IMAGE_NAME    = "sathish1102/bloggingapp"
        IMAGE_TAG     = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout Source') {
            steps {
                git branch: 'main',
                    credentialsId: 'git-cred',
                    url: 'https://github.com/Sathish-11/FullStack-Blogging-App.git'
            }
        }

        stage('Build & Test') {
            steps {
                sh "mvn clean verify"
            }
        }

        stage('Trivy File System Scan') {
            steps {
                sh "trivy fs --scanners vuln --format table -o trivy-fs-report.html ."
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh """
                    sonar-scanner \
                    -Dsonar.projectName=bloggingapp \
                    -Dsonar.projectKey=bloggingapp \
                    -Dsonar.java.binaries=target
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh """
                    docker build \
                    -t ${IMAGE_NAME}:${IMAGE_TAG} \
                    -t ${IMAGE_NAME}:latest .
                    """
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh """
                trivy image \
                --severity HIGH,CRITICAL \
                --format table \
                -o trivy-image-report.html \
                ${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh """
                        docker push ${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${IMAGE_NAME}:latest
                        """
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withKubeConfig(
                    credentialsId: 'k8-cred',
                    clusterName: 'abrahimcse-cluster',
                    namespace: 'webapps',
                    serverUrl: 'https://A6B32A6AF6C9CAF59FB52F47B77E531F.gr7.ap-south-1.eks.amazonaws.com'
                ) {
                    sh "kubectl apply -f deployment-service.yaml"
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withKubeConfig(
                    credentialsId: 'k8-cred',
                    clusterName: 'abrahimcse-cluster',
                    namespace: 'webapps',
                    serverUrl: 'https://A6B32A6AF6C9CAF59FB52F47B77E531F.gr7.ap-south-1.eks.amazonaws.com'
                ) {
                    sh """
                    kubectl get pods -n webapps
                    kubectl get svc -n webapps
                    """
                }
            }
        }
    }

    post {
        always {
            script {
                def status = currentBuild.currentResult

                emailext(
                    subject: "Jenkins | ${env.JOB_NAME} #${env.BUILD_NUMBER} | ${status}",
                    body: """
                        <h3>Pipeline Status: ${status}</h3>
                        <p>Job: ${env.JOB_NAME}</p>
                        <p>Build: ${env.BUILD_NUMBER}</p>
                        <p><a href="${env.BUILD_URL}">View Console Output</a></p>
                    """,
                    to: 'just1sathish@gmail.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-image-report.html'
                )
            }
        }
    }
}
