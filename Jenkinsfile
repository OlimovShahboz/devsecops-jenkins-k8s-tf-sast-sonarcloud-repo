pipeline {
  agent any
  tools {
        maven 'Maven_3_5_2'
    }
  environment {
        REPORT_EMAIL = 'olimovshahboz501@gmail.com'
        // ECR target; the push uses the EC2 instance role, so no AWS keys are stored in Jenkins
        AWS_REGION     = 'eu-north-1'
        AWS_ACCOUNT_ID = '855215820447'
        ECR_REGISTRY   = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        ECR_REPO       = 'buggy-app'
        IMAGE_TAG      = "${env.BUILD_NUMBER}"
        // EKS target (created by eks.tf in the Terraform project); access via the same instance role
        EKS_CLUSTER    = 'buggy-app-eks'
        HELM_VERSION   = 'v3.22.1'
        KUBECONFIG     = "${env.WORKSPACE}/.kube/config"
        // Argo CD deploys the app from Git; Jenkins only installs it once and bumps the image tag
        ARGOCD_CHART_VERSION = '10.14.0'
        ARGOCD_ALLOWED_CIDR  = '132.231.155.33/32' // who may open the Argo CD UI: your public IP
        GITOPS_REPO          = 'github.com/OlimovShahboz/devsecops-jenkins-k8s-tf-sast-sonarcloud-repo.git'
        VALUES_FILE          = 'helm/buggy-app/values.yaml'
    }
   stages{
    stage('CompileandRunSonarAnalysis') {
            steps {
		withCredentials([string(credentialsId: 'sonarcloud-token', variable: 'SONAR_TOKEN')]) {
		sh 'mvn clean verify sonar:sonar -Dsonar.token=$SONAR_TOKEN'
		}
			}
      }

    stage('RunSCAAnalysisUsingSnyk') {
            steps {
		withCredentials([string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')]) {
		sh 'mvn snyk:test@snyk-test'
		sh 'mvn snyk:monitor@snyk-monitor'
		}
		// Turn the JSON results into a readable HTML report
		sh 'npx --yes snyk-to-html@3.7.11 -i snyk-report.json -o snyk-report.html'
			}
      }

    stage('BuildDockerImage') {
            steps {
		sh 'docker build -t $ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG -t $ECR_REGISTRY/$ECR_REPO:latest .'
			}
      }

    stage('PushImageToECR') {
            steps {
		// Short-lived ECR login token from the instance role, piped straight into docker login
		sh 'aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY'
		sh 'docker push $ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG'
		sh 'docker push $ECR_REGISTRY/$ECR_REPO:latest'
			}
      }

    stage('InstallArgoCD') {
            steps {
		// Helm isn't installed on the Jenkins server: download the pinned release once and verify its checksum
		sh '''
		  HELM="$HOME/bin/helm-$HELM_VERSION"
		  if [ ! -x "$HELM" ]; then
		    TMP=$(mktemp -d)
		    curl -fsSLo "$TMP/helm.tar.gz" "https://get.helm.sh/helm-$HELM_VERSION-linux-amd64.tar.gz"
		    echo "$(curl -fsSL "https://get.helm.sh/helm-$HELM_VERSION-linux-amd64.tar.gz.sha256sum" | cut -d' ' -f1)  $TMP/helm.tar.gz" | sha256sum -c -
		    tar -xzf "$TMP/helm.tar.gz" -C "$TMP"
		    mkdir -p "$HOME/bin" && install -m 0755 "$TMP/linux-amd64/helm" "$HELM"
		    rm -rf "$TMP"
		  fi
		'''
		// kubeconfig for this build only; kubectl and helm authenticate with the instance role via aws eks get-token
		sh 'aws eks update-kubeconfig --region $AWS_REGION --name $EKS_CLUSTER'
		// Install Argo CD only the first time; later builds leave it alone
		sh '''
		  HELM="$HOME/bin/helm-$HELM_VERSION"
		  if ! "$HELM" status argocd --namespace argocd >/dev/null 2>&1; then
		    "$HELM" upgrade --install argocd argo-cd \
		      --repo https://argoproj.github.io/argo-helm --version $ARGOCD_CHART_VERSION \
		      --namespace argocd --create-namespace \
		      --set server.service.type=LoadBalancer \
		      --set "server.service.loadBalancerSourceRanges[0]=$ARGOCD_ALLOWED_CIDR" \
		      --wait --timeout 10m
		  fi
		'''
		// Register buggy-app with Argo CD (kubectl apply is idempotent, so this is safe on every build)
		sh 'kubectl apply -f argocd/buggy-app-application.yaml'
			}
      }

    stage('UpdateHelmImageTag') {
            steps {
		// GitOps: write the new image tag into the chart and push it; Argo CD then deploys that commit
		withCredentials([usernamePassword(credentialsId: 'github-token', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
		sh '''
		  git fetch origin main
		  git checkout -B main origin/main
		  sed -i "s|^  tag: .*|  tag: \\"$IMAGE_TAG\\"|" $VALUES_FILE
		  grep -n "^  tag:" $VALUES_FILE
		  git config user.name "jenkins"
		  git config user.email "jenkins@buggy-app.local"
		  git add $VALUES_FILE
		  git commit -m "Deploy buggy-app:$IMAGE_TAG [skip ci]"
		  git push "https://$GIT_USER:$GIT_TOKEN@$GITOPS_REPO" main
		'''
		}
			}
      }
  }
  post {
    always {
		archiveArtifacts artifacts: 'snyk-report.*', allowEmptyArchive: true
		emailext(
			to: "${REPORT_EMAIL}",
			subject: "Snyk SCA report: ${env.JOB_NAME} #${env.BUILD_NUMBER} (${currentBuild.currentResult})",
			body: "Snyk SCA results for ${env.JOB_NAME} build #${env.BUILD_NUMBER} are attached (HTML and JSON).\nBuild: ${env.BUILD_URL}",
			attachmentsPattern: 'snyk-report.*'
		)
    }
  }
}
