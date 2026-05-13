pipeline {
    agent any

    tools {
        sonarScanner 'SonarScanner'
    }

    stages {
        stage('Checkout Source Code') {
            steps {
                echo 'Checking out source code from GitHub...'
                checkout scm
            }
        }

        stage('SAST Scan - SonarQube') {
            steps {
                echo 'Running SonarQube SAST scan...'
                withSonarQubeEnv('SonarQube') {
                    sh '''
                    sonar-scanner \
                    -Dsonar.projectKey=juice-shop \
                    -Dsonar.projectName="OWASP Juice Shop" \
                    -Dsonar.sources=juice-shop \
                    -Dsonar.exclusions=**/node_modules/**,**/dist/**,**/build/**
                    '''
                }
            }
        }

        stage('SCA Scan - Dependency-Check') {
            steps {
                echo 'SCA scan stage placeholder. To be completed by SCA member.'
            }
        }

        stage('Deploy Juice Shop Target') {
            steps {
                echo 'Preparing Juice Shop target environment...'
                sh 'docker stop juice-shop || true'
                sh 'docker rm juice-shop || true'
                sh 'docker run --rm -d -p 3000:3000 --name juice-shop bkimminich/juice-shop'
                sleep time: 15, unit: 'SECONDS'
            }
        }

        stage('DAST Scan - OWASP ZAP') {
            steps {
                echo 'Running OWASP ZAP baseline scan...'
                sh 'docker rm zap_scan || true'
                sh 'docker volume rm zap_temp || true'

                sh 'docker run -u root --name zap_scan -v zap_temp:/zap/wrk -t zaproxy/zap-stable zap-baseline.py -t http://host.docker.internal:3000 -r zap_report.html || true'

                sh 'docker cp zap_scan:/zap/wrk/zap_report.html zap_report.html || true'

                sh 'docker rm zap_scan || true'
                sh 'docker volume rm zap_temp || true'
            }
        }
    }

    post {
        always {
            echo 'Archiving reports and cleaning up...'
            archiveArtifacts artifacts: 'zap_report.html', allowEmptyArchive: true
            sh 'docker stop juice-shop || true'
        }
    }
}