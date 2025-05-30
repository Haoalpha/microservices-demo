pipeline {
    agent any
    environment {
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_CREDENTIALS_ID = 'docker-hub-credentials'
        IMAGE_TAG = "${env.BUILD_ID}-${new Date().format('yyyyMMddHHmmss')}"
    }
    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/Haoalpha/microservices-demo.git', branch: 'main'
            }
        }
        stage('Build Docker Images') {
            steps {
                script {
                    def services = [
                        'src/frontend',
                        'src/productcatalogservice'
                    ]
                    for (service in services) {
                        dir(service) {
                            bat "docker build -t ${DOCKER_REGISTRY}/tumachieu/${service.split('/').last()}:${IMAGE_TAG} ."
                        }
                    }
                }
            }
        }
        stage('Test') {
            steps {
                script {
                    // Test frontend on port 8081
                    bat """
                        docker run -d --name test-frontend -p 8081:8080 ${DOCKER_REGISTRY}/tumachieu/frontend:${IMAGE_TAG}
                        powershell -Command "Start-Sleep -Seconds 20"
                        powershell -Command "try { Invoke-WebRequest -Uri http://localhost:8081 -UseBasicParsing -TimeoutSec 10 } catch { exit 1 }"
                        docker rm -f test-frontend
                    """
                    // Test productcatalogservice on port 3550
                    bat """
                        docker run -d --name test-productcatalog -p 3550:3550 ${DOCKER_REGISTRY}/tumachieu/productcatalogservice:${IMAGE_TAG}
                        powershell -Command "Start-Sleep -Seconds 20"
                        powershell -Command "try { Invoke-WebRequest -Uri http://localhost:3550 -UseBasicParsing -TimeoutSec 10 } catch { exit 1 }"
                        docker rm -f test-productcatalog
                    """
                }
            }
        }
        stage('Push Images') {
            steps {
                script {
                    docker.withRegistry("https://${DOCKER_REGISTRY}", DOCKER_CREDENTIALS_ID) {
                        def services = [
                            'src/frontend',
                            'src/productcatalogservice'
                        ]
                        for (service in services) {
                            def imageName = "${DOCKER_REGISTRY}/tumachieu/${service.split('/').last()}:${IMAGE_TAG}"
                            bat "docker push ${imageName}"
                        }
                    }
                }
            }
        }
        stage('Deploy') {
            steps {
                script {
                    dir('k8s-manifests') {
                        bat """
                            powershell -Command "(Get-Content docker-compose.yml) -replace '{{IMAGE_TAG}}', '${IMAGE_TAG}' | Set-Content docker-compose.yml"
                            docker-compose up -d
                        """
                    }
                }
            }
        }
    }
    post {
        always {
            bat 'docker system prune -f'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}