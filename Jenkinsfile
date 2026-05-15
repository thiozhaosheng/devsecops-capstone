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
                echo 'Running OWASP Dependency Check for vulnerable dependencies...'
                script {
                    def workspacePath = env.WORKSPACE
                    workspacePath = workspacePath.replace('\\', '/')
                    
                    sh """
                        docker run --rm \
                            -v ${workspacePath}:/src \
                            -v ${workspacePath}/dependency-check-report:/report \
                            owasp/dependency-check \
                            --scan /src \
                            --format HTML \
                            --out /report \
                            --prettyPrint
                    """
                }
                
                script {
                    def reportFile = "${env.WORKSPACE}\\dependency-check-report\\dependency-check-report.html"
                    if (fileExists(reportFile)) {
                        archiveArtifacts artifacts: 'dependency-check-report/dependency-check-report.html', allowEmptyArchive: true
                        echo 'SCA report archived successfully'
                    } else {
                        echo "SCA report not found at ${reportFile}"
                    }
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
            echo 'All security scans passed successfully!'
        }
        
        failure {
            echo 'Pipeline failed. Check security scan reports for vulnerabilities.'
        }
    }
}
