pipeline {
    agent any
    tools { 
        maven 'Maven_3_8_7'  
    }
    environment {
        SONAR_TOKEN = credentials('sonar-token')
        SNYK_TOKEN = credentials('SNYK_TOKEN')
        DOCKER_CREDENTIALS_ID = 'dockerlogin'
        KUBE_CREDENTIALS_ID = 'kubelogin'
        DOCKER_IMAGE_NAME = 'sarojamarraj/asg'
        DOCKER_REGISTRY = 'https://registry.hub.docker.com'
        SONAR_PROJECT_KEY = 'asbuggywebapp6'
        SONAR_ORGANIZATION = 'asbuggywebapp6'
    }
    stages {
        stage('Run Gitleaks') {
            steps {
                script {
                    // Run Gitleaks and generate JSON report, but continue regardless of exit status
                    sh(script: "gitleaks detect --source . --report-path=gitleaks-report.json", returnStatus: true)
                }
            }
        }
        
        stage('Compile and Run Sonar Analysis White box') {
            steps {    
                sh "mvn clean verify sonar:sonar -Dsonar.projectKey=${SONAR_PROJECT_KEY} -Dsonar.organization=${SONAR_ORGANIZATION} -Dsonar.host.url=https://sonarcloud.io -Dsonar.token=${SONAR_TOKEN}"
            }
        }

        stage('Run SCA Analysis Using Snyk') {
            steps {        
                withCredentials([string(credentialsId: 'SNYK_TOKEN', variable: 'SNYK_TOKEN')]) {
                    sh 'mvn snyk:test -fn'
                }
            }
        }
        
        stage('Build Docker Image') { 
            steps { 
                script {
                    app = docker.build("${DOCKER_IMAGE_NAME}")
                }
            }
        }
        
        stage('Push image to repo') {
            steps {
                script {
                    docker.withRegistry("${DOCKER_REGISTRY}", "${DOCKER_CREDENTIALS_ID}") {
                        app.push("latest")
                    }
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                withKubeConfig([credentialsId: "${KUBE_CREDENTIALS_ID}"]) {
                    sh('kubectl delete all --all -n devsecops')
                    sh('kubectl apply -f deployment.yaml --namespace=devsecops')
                }
            }
        }
        
        stage('Waiting for deployment to stabilize') {
            steps {
                sh 'pwd; sleep 60; echo "Application has been deployed on K8S"'
            }
        }

        stage('DAST BlackBox') {
            steps {
                withKubeConfig([credentialsId: "${KUBE_CREDENTIALS_ID}"]) {
                    script {
                        def serviceUrl = sh(script: 'kubectl get services/asg --namespace=devsecops -o json | jq -r ".spec.clusterIP"', returnStdout: true).trim()
                        echo "Service URL: ${serviceUrl}"
                        sh "zap.sh -cmd -port 8090 -quickurl http://${serviceUrl} -quickprogress -quickout ${WORKSPACE}/zap_report.html"
                    }
                }
                archiveArtifacts artifacts: 'zap_report.html'
            }
        }
        
        
    }
    
    post {
        always {
            // Archive both the Gitleaks JSON report for inspection
            archiveArtifacts artifacts: 'gitleaks-report.json', allowEmptyArchive: true
        }
        failure {
            echo "Build failed, check reports for detected issues!"
        }
    }
}
