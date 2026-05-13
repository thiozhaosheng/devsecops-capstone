pipeline {
    agent any
    
    stages {
        stage('Deploy Juice Shop Target') {
            steps {
                echo 'Preparing the target environment...'
                sh 'docker stop juice-shop || true'
                sh 'docker rm juice-shop || true'
                
                sh 'docker run --rm -d -p 3000:3000 --name juice-shop bkimminich/juice-shop'
                sleep time: 15, unit: 'SECONDS'
            }
        }
        
        stage('DAST Security Scan (OWASP ZAP)') {
            steps {
                echo 'Initiating OWASP ZAP automated attack...'
                sh 'docker rm zap_scan || true'
                sh 'docker volume rm zap_temp || true'
                
                // THE FIX: Added "-u root" so ZAP has permission to save the HTML report
                sh 'docker run -u root --name zap_scan -v zap_temp:/zap/wrk -t zaproxy/zap-stable zap-baseline.py -t http://host.docker.internal:3000 -r zap_report.html || true'
                
                sh 'docker cp zap_scan:/zap/wrk/zap_report.html zap_report.html || true'
                
                sh 'docker rm zap_scan || true'
                sh 'docker volume rm zap_temp || true'
            }
        }
    }
    
    post {
        always {
            echo 'Saving evidence and cleaning up...'
            archiveArtifacts artifacts: 'zap_report.html', allowEmptyArchive: true
            sh 'docker stop juice-shop || true'
        }
    }
}