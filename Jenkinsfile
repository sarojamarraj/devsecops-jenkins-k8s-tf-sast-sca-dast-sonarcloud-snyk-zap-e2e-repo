pipeline {
    agent any
    tools { 
        maven 'Maven_3_8_7'  
    }
    stage('Run Gitleaks') {
            steps {
                script {
                    // Run Gitleaks and generate JSON report, but continue regardless of exit status
                    sh(script: "gitleaks detect --source . --report-path=gitleaks-report.json", returnStatus: true)
                }
            }
        }
    stages {
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
                withDockerRegistry([credentialsId: "dockerlogin", url: ""]) {
                    script {
                        app = docker.build("sarojamarraj/asg:latest")
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                withDockerRegistry([credentialsId: "dockerlogin", url: ""]) {
                    script {
                        app.push("sarojamarraj/asg:latest")
                    }
                }
            }
        }
    }
}
