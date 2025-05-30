pipeline { 
    agent { 
        kubernetes { 
            defaultContainer 'jnlp'
            yaml """
apiVersion: v1 
kind: Pod 
metadata: 
  name: jenkins
spec: 
  volumes: 
    - name: docker-sock 
      hostPath: 
        path: /var/run/docker.sock 
  containers: 
  - name: docker 
    image: bitnami/kubectl:latest 
    imagePullPolicy: IfNotPresent 
    command: 
    - cat 
    tty: true 
    volumeMounts:   
    - mountPath: /var/run/docker.sock 
      name: docker-sock 
""" 
        }
    }

    environment {
        APP_NAME = "nginx-dash"
        IMAGE_TAG = "komall6/nginx-dash:build-${env.BUILD_NUMBER}"
    }

    stages {
        stage('Clone GitHub Repo') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/Dhanvikah/Demo.git'
            }
        }

        stage('Docker Login') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'Docker-creds', 
                    usernameVariable: 'username', 
                    passwordVariable: 'passwd'
                )]) {
                    container('docker') {
                        sh """
                        docker login -u $username -p $passwd
                        """
                    }
                }
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
                container('docker') {
                    sh "docker build -t ${env.IMAGE_TAG} ."
                }
            }
        }

        stage('Push Docker Image to Docker Hub') {
            steps {
                container('docker') {
                    sh "docker push ${env.IMAGE_TAG}"
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                container('docker') {
                    script {
                        sh """
                        sed -i 's|komall6/nginx-dash:build-REPLACE_TAG|${env.IMAGE_TAG}|' k8s-file.yaml
                        kubectl apply -f k8s-file.yaml
                        """
                    }
                }
            }
        }

        stage('ArgoCD Login & Sync') {
            steps {
                container('docker') {
                    withCredentials([usernamePassword(
                        credentialsId: 'argocd-creds',
                        usernameVariable: 'ARGO_USER',
                        passwordVariable: 'ARGO_PASS'
                    )]) {
                        sh '''
                        curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
                        chmod +x argocd && mv argocd /usr/local/bin/argocd

                        argocd login localhost:8080 --username $ARGO_USER --password $ARGO_PASS --insecure
                        argocd app sync $APP_NAME
                        '''
                    }
                }
            }
        }
    }
}
