pipeline {
  agent any

  tools {
    jdk 'jdk17'
    maven 'maven3'
  }

 environment {
        SCANNER_HOME = tool 'sonar-scanner'
        JAVA_HOME = tool 'jdk17'
        PATH = "${JAVA_HOME}/bin:${env.PATH}"
    }

  stages {
    stage('Git Checkout') {
      steps {
        git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/patilpavan8333/BoardGame-Project.git'
      }
    }

    stage('Compile') {
      steps {
        sh "echo JAVA_HOME=$JAVA_HOME"
        sh "mvn compile"
      }
    }

    stage('Test') {
      steps {
        sh "mvn test"
      }
    }

    stage('File System Scan') {
      steps {
        sh "trivy fs --format table -o trivy-fs-report.html ."
      }
    }

    stage('Build') {
      steps {
        sh "mvn package"
      }
      
    }
    stage('SonarQube Analysis') {
    steps {
        withSonarQubeEnv('sonarqube') {
            sh '''
                $SCANNER_HOME/bin/sonar-scanner \
                  -Dsonar.projectName=BoardGame \
                  -Dsonar.projectKey=BoardGame \
                  -Dsonar.java.binaries=. \
                  -Dsonar.host.url=http://52.221.183.179:9000
            '''
        }
    }
}
        
        stage('Quality Gate') {
            steps {
                script {
                  waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token' 
                }
            }
        }

   stage('Build & Tag Docker Image') {
    steps {
        script {
            withDockerRegistry([
                url: 'https://index.docker.io/v1/',
                credentialsId: 'dockerhub-credentials'
            ]) {
                sh 'docker build -t patil8333/boardshack:latest .'
            }
        }
    }
}

    stage('Push Docker Image') {
      steps {
        script {
          withDockerRegistry(credentialsId: 'dockerhub-credentials', toolName: 'docker') {
              sh "docker push patil8333/boardshack:latest"
          }
        }
      }
    }

    stage('Deploy To Kubernetes') {
      steps {
        withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'k8-cred', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://172.31.8.146:6443') {
            sh 'kubectl create namespace webapps --dry-run=client -o yaml | kubectl apply -f -'
            sh "kubectl apply -f deployment-service.yaml"
        }
      }
    }

    stage('Verify the Deployment') {
      steps {
        withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'k8-cred', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://172.31.8.146:6443') {
            sh '''
                kubectl get pods -n webapps
                kubectl get svc -n webapps
                 # Install Helm locally in Jenkins workspace (no sudo required)
                if ! command -v helm > /dev/null; then
                  curl -fsSL https://get.helm.sh/helm-v3.21.3-linux-amd64.tar.gz -o helm.tar.gz
                  tar -xzf helm.tar.gz
                  mv linux-amd64/helm ./helm
                  chmod +x ./helm
                  export PATH=$PWD:$PATH
                fi

                # Add Prometheus Helm repo
                ./helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
                ./helm repo update

                # Install Prometheus stack
                ./helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
                  --namespace monitoring \
                  --create-namespace
                  
                    kubectl patch svc monitoring-kube-prometheus-prometheus -n monitoring \
                    -p '{"spec":{"type":"LoadBalancer"}}'

                    kubectl get svc -n monitoring
                    '''
        }
      }
    }
  }

  post {
    always {
      script {
        def jobName = env.JOB_NAME
        def buildNumber = env.BUILD_NUMBER
        def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
        def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'

        def body = """
                <html>
                <body>
                <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                <h2>${jobName} - Build ${buildNumber}</h2>
                <div style="background-color: ${bannerColor}; padding: 10px;">
                <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
                </div>
                <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                </div>
                </body>
                </html>
            """

        emailext (
          subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
          body: body,
          to: 'patilpavan8333@gmail.com',
          from: 'jenkins@example.com',
          replyTo: 'jenkins@example.com',
          mimeType: 'text/html',
          attachmentsPattern: 'trivy-image-report.html'
        )
      }
    }
  }
}
