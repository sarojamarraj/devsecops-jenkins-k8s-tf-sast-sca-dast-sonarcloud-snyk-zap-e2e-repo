pipeline {
    agent any
    tools { 
        maven 'Maven_3_8_7'  
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
        
        stage('Compile and Run Sonar Analysis') {
            steps {    
                sh 'mvn clean verify sonar:sonar -Dsonar.projectKey=asbuggywebapp6 -Dsonar.organization=asbuggywebapp6 -Dsonar.host.url=https://sonarcloud.io -Dsonar.token=0ab485f7b37004e64bf1df4e8ca4c5be980cb930'
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
                    app = docker.build("sarojamarraj/asg")
                }
            }
        }
        
        stage('Push image') {
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'dockerlogin') {
                        app.push("latest")
                    }
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                sh 'kubectl apply -f deployment.yaml'
            }
        }

        stage('Run DAST Using ZAP') {
            steps {
                withKubeConfig([credentialsId: 'kubelogin']) {
                    script {
                        def serviceUrl = sh(script: 'kubectl get services/asg --namespace=devsecops -o json | jq -r ".status.loadBalancer.ingress[] | .hostname"', returnStdout: true).trim()
                        sh "zap.sh -cmd -port 8090 -quickurl http://${serviceUrl} -quickprogress -quickout ${WORKSPACE}/zap_report.html"
                    }
                    archiveArtifacts artifacts: 'zap_report.html'
                }
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
