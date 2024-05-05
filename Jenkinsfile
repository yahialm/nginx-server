pipeline {
  agent {
    kubernetes {
      cloud "kubernetes"
      yamlFile "agent-build.yaml"
    }
  }
  
  stages {
    stage('Build and push to gcr') {
      steps {
        container("gcloud-builder") {
          sh 'git config --global --add safe.directory /home/jenkins/agent/workspace/nginx-server'
          script {
            // Generate a unique tag based on the commit hash
            def commitHash = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
            env.dockerTag = "dev-commit-${commitHash}-${BUILD_NUMBER}"
            env.gkeClusterName = 'cluster-1'
            env.Zone = 'us-central1-c'
            env.gkeProject = 'booming-being-413416'
            
            sh "docker build -t nginx-server:${env.dockerTag} ."
            
            withCredentials([file(credentialsId: 'gcr-id', variable: 'SERVICE_ACCOUNT_KEY')]) {
              sh 'gcloud auth activate-service-account --key-file=$SERVICE_ACCOUNT_KEY'
              sh "gcloud container clusters get-credentials ${env.gkeClusterName} --zone ${env.Zone} --project ${env.gkeProject}"
              sh "gcloud auth configure-docker"
            }
            
            env.gcrImage = "gcr.io/${env.gkeProject}/nginx-server:${env.dockerTag}"
            sh "docker tag nginx-server:${env.dockerTag} ${env.gcrImage}"
            sh "docker push ${env.gcrImage}"
          }
        }
      }
    }

    stage('Deploy to GKE') {
      steps {
        container("gcloud-builder") {
          script {
            // Check if deployment exists
            def deploymentExists = sh(returnStatus: true, script: "kubectl get deployment nginx-server -n nginx-server")
            if (deploymentExists != 0) {
              // If deployment doesn't exist, create it
              sh "kubectl create deployment nginx-server --image=${env.gcrImage} -n nginx-server"
            } else {
              // If deployment exists, update its image
              sh "kubectl set image deployment/nginx-server nginx-server=${env.gcrImage} -n nginx-server"
            }
            // Delete existing Service if it exists
            sh "kubectl delete service nginx-server -n nginx-server --ignore-not-found"

            // Expose the Deployment as a LoadBalancer Service
            sh "kubectl expose deployment nginx-server --type=LoadBalancer --port=80 --target-port=80 -n nginx-server"
          }
        }
      }
    } 
  }

  post {
    always {
      // Clean up the agent pod after the job completes
      cleanWs()
      deleteDir()
    }
  }
}
