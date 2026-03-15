pipeline {
    agent any

    tools {
        dockerTool 'docker'
    }

    environment {
        PATH = "${tool 'docker'}/bin:${env.PATH}"
        DOCKER_IMAGE = "iiitkabel/jenkins_automation_wine_prediction:latest"
        CONTAINER_NAME = "wine_inference_test"
        
        // THE CHEAT CODE: 172.17.0.1 points Jenkins to your laptop's exposed ports!
        API_URL = "http://172.17.0.1:8000"
    }

    stages {
        stage('Pull Image') {
            steps {
                sh "docker pull ${DOCKER_IMAGE}"
            }
        }

        stage('Run Container') {
            steps {
                sh "docker rm -f ${CONTAINER_NAME} || true"
                // Run normally, exposing to the laptop
                sh "docker run -d -p 8000:8000 --name ${CONTAINER_NAME} ${DOCKER_IMAGE}"
            }
        }

        stage('Wait for Service Readiness') {
            steps {
                timeout(time: 1, unit: 'MINUTES') {
                    retry(12) {
                        sh "sleep 5"
                        // This will now hit 172.17.0.1 instead of localhost
                        sh "curl -f -s ${API_URL}/docs > /dev/null"
                    }
                }
            }
        }

        stage('Send Valid Request') {
            steps {
                script {
                    def response = sh(script: "curl -s -w '\\n%{http_code}' -X POST ${API_URL}/predict -H 'Content-Type: application/json' -d @inputs/valid.json", returnStdout: true).trim()
                    
                    def lines = response.split('\n')
                    def body = lines[0]
                    def status = lines[1]
                    
                    echo "API Response: ${body}"
                    echo "HTTP Status: ${status}"
                    
                    if (status != "200") { error("Validation Failed") }
                }
            }
        }

        stage('Send Invalid Request') {
            steps {
                script {
                    def response = sh(script: "curl -s -w '\\n%{http_code}' -X POST ${API_URL}/predict -H 'Content-Type: application/json' -d @inputs/invalid.json", returnStdout: true).trim()
                    
                    def lines = response.split('\n')
                    def body = lines[0]
                    def status = lines[1]
                    
                    echo "API Error Response: ${body}"
                    echo "HTTP Error Status: ${status}"
                    
                    if (status == "200") { error("Validation Failed") }
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
        always {
            sh "docker rm -f ${CONTAINER_NAME} || true"
        }
    }
}
