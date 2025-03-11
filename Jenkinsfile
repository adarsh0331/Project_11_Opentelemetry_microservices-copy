pipeline {
    agent any

    stages {
        stage('Deploy To Kubernetes') {
            steps {
                withKubeCredentials(kubectlCredentials: [[caCertificate: '', clusterName: 'eks', contextName: '', credentialsId: 'k8s-token', namespace: 'microservice', serverUrl: 'https://81E3DBA5BF90B20CCB5DF15CC8443E8B.gr7.us-east-1.eks.amazonaws.com']]){
                    sh "kubectl apply -f deployment-service.yml"
                    
                }
            }
        }
        
        stage('verify Deployment') {
            steps {
                withKubeCredentials(kubectlCredentials: [[caCertificate: '', clusterName: 'eks', contextName: '', credentialsId: 'k8s-token', namespace: 'microservice', serverUrl: 'https://81E3DBA5BF90B20CCB5DF15CC8443E8B.gr7.us-east-1.eks.amazonaws.com']]){
                    sh "kubectl get svc -n microservices"
                }
            }
        }
    }
}
