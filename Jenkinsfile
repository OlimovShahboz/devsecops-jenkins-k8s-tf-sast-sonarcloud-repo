pipeline {
  agent any
  tools {
        maven 'Maven_3_5_2'
    }
   stages{
    stage('CompileandRunSonarAnalysis') {
            steps {
		withCredentials([string(credentialsId: 'sonarcloud-token', variable: 'SONAR_TOKEN')]) {
		sh 'mvn clean verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=testoneado -Dsonar.organization=testoneado -Dsonar.host.url=https://sonarcloud.io -Dsonar.token=$SONAR_TOKEN'
		}
			}
        }
  }
}
