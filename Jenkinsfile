pipeline {
    agent any

    stages {
        stage('Deploy To Kubernetes') {
            steps {
                   withKubeConfig(caCertificate: '', clusterName: 'eks', contextName: '', credentialsId: 'k8s-cred', namespace: 'ms', restrictKubeConfigAccess: false, serverUrl: 'https://B775572DFEC747C548A81ADFA31189DC.gr7.us-east-1.eks.amazonaws.com') {
                    sh "kubectl apply -f deployment-service.yml"
                    
                }
            }
        }
        
        stage('verify Deployment') {
            steps {
                  withKubeConfig(caCertificate: '', clusterName: 'eks', contextName: '', credentialsId: 'k8s-cred', namespace: 'ms', restrictKubeConfigAccess: false, serverUrl: 'https://B775572DFEC747C548A81ADFA31189DC.gr7.us-east-1.eks.amazonaws.com') {  
                  sh "kubectl get svc -n ms"
                }
            }
        }
    }
}
