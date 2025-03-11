pipeline {
    agent any

    stages {
        stage('Deploy To Kubernetes') {
            steps {
                withKubeCredentials(kubectlCredentials: [[caCertificate: '', clusterName: 'EKS', contextName: '', credentialsId: 'k8-token', namespace: 'webapps', serverUrl: 'https://42D858905D8ED0CF9B1BB70C8CA4A755.gr7.us-east-1.eks.amazonaws.com']]) {
                    sh "kubectl apply -f deployment-service.yml"
                    
                }
            }
        }
        
        stage('verify Deployment') {
            steps {
                withKubeCredentials(kubectlCredentials: [[caCertificate: '', clusterName: 'EKS', contextName: '', credentialsId: 'k8-token', namespace: 'webapps', serverUrl: 'https://42D858905D8ED0CF9B1BB70C8CA4A755.gr7.us-east-1.eks.amazonaws.com']]) {
                    sh "kubectl get svc -n webapps"
                }
            }
        }
    }
}
