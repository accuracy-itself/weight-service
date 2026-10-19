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
                cat <<EOF | kubectl apply -f -
                apiVersion: batch/v1
                kind: Job
                metadata:
                  name: kaniko-build-${BUILD_NUMBER}
                  namespace: jenkins
                spec:
                  backoffLimit: 0
                  template:
                    spec:
                      restartPolicy: Never
                      containers:
                      - name: kaniko
                        image: gcr.io/kaniko-project/executor:latest
                        args:
                        - "--dockerfile=Dockerfile"
                        - "--context=dir:///workspace"
                        - "--destination=${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
                        - "--destination=${REGISTRY}/${IMAGE_NAME}:latest"
                        - "--insecure"
                        - "--skip-tls-verify"
                        volumeMounts:
                        - name: workspace
                          mountPath: /workspace
                        - name: kaniko-secret
                          mountPath: /kaniko/.docker
                      volumes:
                      - name: workspace
                        persistentVolumeClaim:
                          claimName: jenkins-pvc # Используем ваш существующий PVC, где лежит код
                      - name: kaniko-secret
                        secret:
                          secretName: kaniko-registry-secret
                          items:
                          - key: .dockerconfigjson
                            path: config.json
                EOF
                """

                sh "kubectl wait --for=condition=complete job/kaniko-build-${BUILD_NUMBER} -n jenkins --timeout=300s"
            }
            post {
                always {
                    sh "kubectl delete job kaniko-build-${BUILD_NUMBER} -n jenkins --ignore-not-found"
                }
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