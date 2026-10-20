pipeline {
    agent any

    environment {
        REGISTRY = 'registry.kube-system.svc.cluster.local:5000'
        IMAGE_NAME = 'weight-service'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Build & Test') {
            steps {
                sh 'mvn clean test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Package') {
            steps {
                sh 'mvn package -DskipTests'
            }
        }

        stage('Kaniko Build & Push') {
            steps {
                sh """
                    sed -e 's|IMAGE_TAG_PLACEHOLDER|${BUILD_NUMBER}|g' \\
                        -e 's|REGISTRY_PLACEHOLDER|${REGISTRY}|g' \\
                        -e 's|IMAGE_NAME_PLACEHOLDER|${IMAGE_NAME}|g' \\
                        k8s/kaniko-job.yaml > /tmp/kaniko-job-${BUILD_NUMBER}.yaml
                    kubectl apply -f /tmp/kaniko-job-${BUILD_NUMBER}.yaml
                """
                sh "kubectl wait --for=condition=complete job/kaniko-build-${BUILD_NUMBER} -n jenkins --timeout=300s"
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh """
                sed -i "s|image: .*|image: ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}|g" k8s/deployment.yaml
                kubectl apply -f k8s/deployment.yaml
                kubectl rollout restart deployment/weight-service
                """
            }
        }
    }

    post {
        failure {
            echo 'Pipeline failed, no deploy.'
        }
    }
}