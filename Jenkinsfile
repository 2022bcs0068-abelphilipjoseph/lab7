pipeline {
    agent any

    tools {
        dockerTool 'docker'
    }

    environment {
        PATH = "${tool 'docker'}/bin:${env.PATH}"
        DOCKER_IMAGE = "iiitkabel/jenkins_automation_wine_prediction:latest"
        CONTAINER_NAME = "wine_inference_test"
        NETWORK_NAME = "jenkins_ml_net"
        
        // THE FIX: Using Docker DNS! Because they share a network, Jenkins can just use the container's name.
        API_URL = "http://${CONTAINER_NAME}:8000"
    }

    stages {
        // Stage 0: Network Setup
        stage('Setup Network') {
            steps {
                echo "Creating a shared Docker network..."
                sh "docker network create ${NETWORK_NAME} || true"
                // Attach Jenkins itself to this network
                sh "docker network connect ${NETWORK_NAME} 2022bcs0068_jenkins || true"
            }
        }

        // Stage 1: Pull Image
        stage('Pull Image') {
            steps {
                echo "Pulling the inference Docker image from Docker Hub..."
                sh "docker pull ${DOCKER_IMAGE}"
            }
        }

        // Stage 2: Run Container
        stage('Run Container') {
            steps {
                echo "Starting the container on the shared network..."
                sh "docker rm -f ${CONTAINER_NAME} || true"
                // Run the API container on the shared network
                sh "docker run -d --network ${NETWORK_NAME} --name ${CONTAINER_NAME} ${DOCKER_IMAGE}"
            }
        }

        // Stage 3: Wait for Service Readiness
        stage('Wait for Service Readiness') {
            steps {
                echo "Waiting until the API responds to ${API_URL}/docs..."
                timeout(time: 1, unit: 'MINUTES') {
                    retry(12) {
                        sh "sleep 5"
                        sh "curl -f -s ${API_URL}/docs > /dev/null"
                    }
                }
                echo "API is up and running!"
            }
        }

        // Stage 4: Send Valid Request
        stage('Send Valid Request') {
            steps {
                echo "Sending a request using the prepared valid input..."
                script {
                    def response = sh(script: "curl -s -w '\\n%{http_code}' -X POST ${API_URL}/predict -H 'Content-Type: application/json' -d @inputs/valid.json", returnStdout: true).trim()
                    
                    def lines = response.split('\n')
                    def body = lines[0]
                    def status = lines[1]
                    
                    echo "API Response: ${body}"
                    echo "HTTP Status: ${status}"
                    
                    if (status != "200") {
                        error("Validation Failed: HTTP status code is not 200")
                    }
                    
                    def json = readJSON text: body
                    if (json.prediction == null || !(json.prediction instanceof Number)) {
                        error("Validation Failed: Prediction field is missing or not numeric")
                    }
                    echo "Valid request checks passed!"
                }
            }
        }

        // Stage 5: Send Invalid Request
        stage('Send Invalid Request') {
            steps {
                echo "Sending malformed or incomplete input..."
                script {
                    def response = sh(script: "curl -s -w '\\n%{http_code}' -X POST ${API_URL}/predict -H 'Content-Type: application/json' -d @inputs/invalid.json", returnStdout: true).trim()
                    
                    def lines = response.split('\n')
                    def body = lines[0]
                    def status = lines[1]
                    
                    echo "API Error Response: ${body}"
                    echo "HTTP Error Status: ${status}"
                    
                    if (status == "200") {
                        error("Validation Failed: API did not return an error for invalid input")
                    }
                    echo "Invalid request checks passed! Error message is meaningful."
                }
            }
        }

        // Stage 6: Stop Container
        stage('Stop Container') {
            steps {
                echo "Stopping and removing the container..."
                sh "docker rm -f ${CONTAINER_NAME}"
            }
        }
    }
    
    // Stage 7: Pipeline Result and Cleanup
    post {
        success {
            echo "Pipeline Result: SUCCESS. All validation tests passed."
        }
        failure {
            echo "Pipeline Result: FAILED. Fetching API Container Logs for debugging:"
            // IF it fails, this will dump the exact error from FastAPI into your Jenkins console!
            sh "docker logs ${CONTAINER_NAME} || true"
        }
        always {
            // Ensure no leftover running containers
            sh "docker rm -f ${CONTAINER_NAME} || true"
        }
    }
}
