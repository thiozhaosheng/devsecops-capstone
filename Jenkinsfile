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

        stage('SCA - OWASP Dependency Check') {
            steps {
                echo 'Preparing dependency files for SCA scan...'

                dir('juice-shop') {
                    sh 'npm install --package-lock-only --ignore-scripts'
                }

                sh 'mkdir -p dependency-check-report'

                dependencyCheck additionalArguments: '''
                    --scan ./juice-shop
                    --format HTML
                    --format XML
                    --out ./dependency-check-report
                    --prettyPrint
                    --nvdApiKey YOUR_API_KEY
                ''', odcInstallation: 'SCA-DependencyCheck'

                dependencyCheckPublisher pattern: 'dependency-check-report/dependency-check-report.xml',
                                         stopBuild: false
            }

            post {
                always {
                    archiveArtifacts artifacts: 'dependency-check-report/*.*',
                                     allowEmptyArchive: true
                }
            }
        }

        stage('Deploy Juice Shop Target') {
            steps {
                echo 'Preparing the target environment...'
                sh 'docker stop juice-shop || true'
                sh 'docker rm juice-shop || true'
                sh 'docker run --rm -d -p 3000:3000 --name juice-shop bkimminich/juice-shop'
                sleep time: 15, unit: 'SECONDS'
                echo 'Juice Shop is now running at http://localhost:3000'
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

                archiveArtifacts artifacts: 'zap_report.html', allowEmptyArchive: true
            }
        }
    }

    post {
        always {
            echo 'Pipeline completed. Cleaning up...'
            sh 'docker stop juice-shop || true'
            echo 'Artifacts available: SCA report, DAST report'
        }

        success {
            echo 'Pipeline completed successfully. Security scan reports are available as artifacts.'
        }

        failure {
            echo 'Pipeline failed due to execution error. Check console logs and security scan reports.'
        }
    }
}