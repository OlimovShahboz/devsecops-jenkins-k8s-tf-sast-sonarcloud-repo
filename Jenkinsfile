pipeline {
  agent any
  tools {
        maven 'Maven_3_5_2'
    }
   stages{
    stage('CompileandRunSonarAnalysis') {
            steps {
		withCredentials([string(credentialsId: 'sonarcloud-token', variable: 'SONAR_TOKEN')]) {
		sh 'mvn clean verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=OlimovShahboz_devsecops-jenkins-k8s-tf-sast-sonarcloud-repo -Dsonar.organization=olimovshahboz -Dsonar.host.url=https://sonarcloud.io -Dsonar.token=$SONAR_TOKEN'
		}
			}
      }

    stage('RunSCAAnalysisUsingSnyk') {
            steps {
		withCredentials([string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')]) {
		sh 'mvn snyk:test'
		}
			}
      }
  }
}
