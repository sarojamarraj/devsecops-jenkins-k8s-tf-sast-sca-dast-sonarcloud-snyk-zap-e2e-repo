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
        
        stage('Compile and Run Sonar Analysis White box') {
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
        
        stage('Push image to repo') {
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
                withKubeConfig([credentialsId: 'kubelogin']) {
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
        withKubeConfig([credentialsId: 'kubelogin']) {
            script {
                def serviceUrl = sh(script: 'kubectl get services/asg --namespace=devsecops -o json | jq -r ".spec.clusterIP"', returnStdout: true).trim()
                echo "Service URL: ${serviceUrl}"
                sh "zap.sh -cmd -port 8090 -quickurl http://${serviceUrl} -quickprogress -quickout ${WORKSPACE}/zap_report.html"
            }
        }
        archiveArtifacts artifacts: 'zap_report.html'
    }
}
post {
        always {
            // Archive both the Gitleaks JSON report for inspection
        }
        failure {
            echo "Build failed, check reports for detected issues!"
        }
    }
    // Archive both the Gitleaks JSON report for inspection
    archiveArtifacts artifacts: 'gitleaks-report.json', allowEmptyArchive: true
} 
}
