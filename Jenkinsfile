pipeline {
    agent any

    tools {
        dockerTool 'docker'
    }

    environment {
        PATH = "${tool 'docker'}/bin:${env.PATH}"
        DOCKER_IMAGE = "iiitkabel/jenkins_automation_wine_prediction:latest"
        CONTAINER_NAME = "wine_inference_test"
    }

    stages {
        stage('Pull Image') {
            steps {
                echo "Pulling the latest image..."
                sh "docker pull ${DOCKER_IMAGE}"
            }
        }

        stage('Run Container') {
            steps {
                sh "docker rm -f ${CONTAINER_NAME} || true"
                sh "docker run -d -p 8000:8000 --name ${CONTAINER_NAME} ${DOCKER_IMAGE}"
                
                script {
                    // THE PROPER FIX: Get the actual IP address of the container dynamically!
                    env.API_IP = sh(script: "docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' ${CONTAINER_NAME}", returnStdout: true).trim()
                    env.API_URL = "http://${env.API_IP}:8000"
                    echo "Container running! API URL dynamically set to: ${env.API_URL}"
                }
            }
        }

        stage('Wait for Service Readiness') {
            steps {
                timeout(time: 1, unit: 'MINUTES') {
                    retry(12) {
                        sh "sleep 5"
                        sh "curl -f -s ${env.API_URL}/docs > /dev/null"
                    }
                }
                echo "API is ready and responding!"
            }
        }

        stage('Send Valid Request') {
            steps {
                script {
                    def response = sh(script: "curl -s -w '\\n%{http_code}' -X POST ${env.API_URL}/predict -H 'Content-Type: application/json' -d @inputs/valid.json", returnStdout: true).trim()
                    
                    def lines = response.split('\n')
                    def body = lines[0]
                    def status = lines[1]
                    
                    echo "API Response: ${body}"
                    echo "HTTP Status: ${status}"
                    
                    if (status != "200") {
                        error("Validation Failed: HTTP status code is not 200")
                    }
                }
            }
        }

        stage('Send Invalid Request') {
            steps {
                script {
                    def response = sh(script: "curl -s -w '\\n%{http_code}' -X POST ${env.API_URL}/predict -H 'Content-Type: application/json' -d @inputs/invalid.json", returnStdout: true).trim()
                    
                    def lines = response.split('\n')
                    def body = lines[0]
                    def status = lines[1]
                    
                    echo "API Error Response: ${body}"
                    echo "HTTP Error Status: ${status}"
                    
                    if (status == "200") {
                        error("Validation Failed: API did not return an error for invalid input")
                    }
                }
            }
        }

        stage('Stop Container') {
            steps {
                sh "docker rm -f ${CONTAINER_NAME}"
            }
        }
    }
    
    post {
        success {
            echo "Pipeline Result: SUCCESS. All validation tests passed."
        }
        failure {
            echo "Pipeline Result: FAILED. Printing container logs for debugging:"
            // If it fails, this prints the exact Python/FastAPI error to your Jenkins console!
            sh "docker logs ${CONTAINER_NAME} || true"
        }
        always {
            sh "docker rm -f ${CONTAINER_NAME} || true"
        }
    }
}
