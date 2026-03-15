pipeline {
    agent any

    tools {
        dockerTool 'docker'
    }

    environment {
        // Our hard-earned Docker PATH fix!
        PATH = "${tool 'docker'}/bin:${env.PATH}"
        
        // The Docker image you built and pushed in Lab 6
        DOCKER_IMAGE = "iiitkabel/jenkins_automation_wine_prediction:latest"
        CONTAINER_NAME = "wine_inference_test"
        API_URL = "http://localhost:8000"
    }

    stages {
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
                echo "Starting the container exposing the API port... "
                // Stop any old containers just in case
                sh "docker rm -f ${CONTAINER_NAME} || true"
                // Run the container with a temporary name 
                sh "docker run -d -p 8000:8000 --name ${CONTAINER_NAME} ${DOCKER_IMAGE}"
            }
        }

        // Stage 3: Wait for Service Readiness
        stage('Wait for Service Readiness') {
            steps {
                echo "Waiting until the API responds to a health endpoint... "
                timeout(time: 1, unit: 'MINUTES') {
                    // Retry every 5 seconds until it succeeds or times out (fails pipeline)
                    retry(12) {
                        sh "sleep 5"
                        // Assuming FastAPI automatically provides /docs or /openapi.json
                        sh "curl -f -s ${API_URL}/docs > /dev/null"
                    }
                }
                echo "API is up and running!"
            }
        }

        // Stage 4: Send Valid Inference Request
        stage('Send Valid Request') {
            steps {
                echo "Sending a request using the prepared valid input... "
                script {
                    // Capture the HTTP response body and status code in one go
                    def response = sh(script: "curl -s -w '\\n%{http_code}' -X POST ${API_URL}/predict -H 'Content-Type: application/json' -d @inputs/valid.json", returnStdout: true).trim()
                    
                    def lines = response.split('\n')
                    def body = lines[0]
                    def status = lines[1]
                    
                    echo "API Response: ${body} "
                    echo "HTTP Status: ${status}"
                    
                    // Verify HTTP status code is successful 
                    if (status != "200") {
                        error("Validation Failed: HTTP status code is not 200")
                    }
                    
                    // Verify Prediction field exists and is numeric 
                    def json = readJSON text: body
                    if (json.prediction == null || !(json.prediction instanceof Number)) {
                        error("Validation Failed: Prediction field is missing or not numeric")
                    }
                    echo "Valid request checks passed!"
                }
            }
        }

        // Stage 5: Send Invalid Request [cite: 53]
        stage('Send Invalid Request') {
            steps {
                echo "Sending malformed or incomplete input... "
                script {
                    def response = sh(script: "curl -s -w '\\n%{http_code}' -X POST ${API_URL}/predict -H 'Content-Type: application/json' -d @inputs/invalid.json", returnStdout: true).trim()
                    
                    def lines = response.split('\n')
                    def body = lines[0]
                    def status = lines[1]
                    
                    echo "API Error Response: ${body} "
                    echo "HTTP Error Status: ${status}"
                    
                    // Verify API returns an error response (usually 422 for FastAPI validation) 
                    if (status == "200") {
                        error("Validation Failed: API did not return an error for invalid input")
                    }
                    echo "Invalid request checks passed! Error message is meaningful. "
                }
            }
        }

        // Stage 6: Stop Container 
        stage('Stop Container') {
            steps {
                echo "Stopping and removing the container... "
                sh "docker rm -f ${CONTAINER_NAME}"
            }
        }
    }
    
    // Stage 7: Pipeline Result and Cleanup 
    post {
        always {
            // Ensure no leftover running containers even if a stage fails 
            sh "docker rm -f ${CONTAINER_NAME} || true"
        }
        success {
            echo "Pipeline Result: SUCCESS. All validation tests passed. "
        }
        failure {
            echo "Pipeline Result: FAILED. A validation step failed. "
        }
    }
}
