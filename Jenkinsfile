pipeline {
  agent any
  tools { 
        maven 'Maven_3_8_7'  
    }
   stages{
    stage('CompileandRunSonarAnalysis') {
            steps {	
		sh 'mvn clean verify sonar:sonar -Dsonar.projectKey=asbuggywebapp6 -Dsonar.organization=asbuggywebapp6 -Dsonar.host.url=https://sonarcloud.io -Dsonar.token=0ab485f7b37004e64bf1df4e8ca4c5be980cb930'
			}
    }

	stage('RunSCAAnalysisUsingSnyk') {
            steps {		
				withCredentials([string(credentialsId: 'SNYK_TOKEN', variable: 'SNYK_TOKEN')]) {
					sh 'mvn snyk:test -fn'
				}
			}
    }

	stage('Build') { 
            steps { 
               withDockerRegistry([credentialsId: "dockerlogin", url: ""]) {
                 script{
                 app =  docker.build("asg")
                 }
               }
            }
    }

   }
}
