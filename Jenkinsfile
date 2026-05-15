pipeline {
    agent any

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
                script {
                    def scannerHome = tool 'SonarScanner'

                    withSonarQubeEnv('SonarQube') {
                        sh """
                        ${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=juice-shop \
                        -Dsonar.projectName="OWASP Juice Shop" \
                        -Dsonar.sources=juice-shop \
                        -Dsonar.exclusions=**/node_modules/**,**/dist/**,**/build/**,**/coverage/**
                        """
                    }
                }
            }
        }

        stage('SCA Scan - Dependency Check') {
            steps {
                echo 'Running OWASP Dependency-Check scan...'

                dependencyCheck additionalArguments: '''
                    --scan . \
                    --format HTML \
                    --out dependency-check-report
                ''',
                odcInstallation: 'DependencyCheck'

                dependencyCheckPublisher pattern: 'dependency-check-report/dependency-check-report.xml'
            }
        }

        stage('Deploy Juice Shop Target') {
            steps {
                echo 'Preparing the target environment...'
                sh 'docker stop juice-shop || true'
                sh 'docker rm juice-shop || true'
                sh 'docker run --rm -d -p 3000:3000 --name juice-shop bkimminich/juice-shop'
                sleep time: 15, unit: 'SECONDS'
            }
        }

        stage('DAST Security Scan - OWASP ZAP') {
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
            echo 'Saving evidence and cleaning up...'

            archiveArtifacts artifacts: 'zap_report.html', allowEmptyArchive: true

            archiveArtifacts artifacts: 'dependency-check-report/**', allowEmptyArchive: true

            sh 'docker stop juice-shop || true'
        }
    }
}
