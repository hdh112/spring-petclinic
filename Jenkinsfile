pipeline {
    agent any

    tools {
        maven 'Maven'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Test') {
            steps {
                sh './mvnw clean package -DskipTests=false'
            }
        }

        stage('SonarQube Code Analysis') {
            steps {
                withCredentials([string(credentialsId: 'SONAR_TOKEN', variable: 'SONAR_TOKEN')]) {
                    sh "./mvnw sonar:sonar -Dsonar.host.url=http://sonarqube:9000 -Dsonar.token=${SONAR_TOKEN}"
                }
            }
        }

        stage('Security Scan (ZAP)') {
            steps {
                script {
                    // Triggers a baseline scan via the container against a test instance 
                    // or directly before production deploy if staging exists.
                    sh "docker exec zap zap-baseline.py -t http://jenkins:8080 -r zap_report.html || true"
                    // Note: '|| true' ensures the pipeline doesn't crash if ZAP finds vulnerabilities, allowing reports to generate.
                }
            }
        }

        stage('Deploy to Production VM via Ansible') {
            steps {
                // Using an ephemeral Docker container to run your playbook 
                // cleanly without needing Ansible installed inside Jenkins
                sh """
                    docker run --rm \
                      -v \$(pwd)/ansible:/ansible \
                      -v /var/jenkins_home/.ssh:/root/.ssh:ro \
                      --network petclinic-net \
                      willhallonline/ansible:latest \
                      ansible-playbook -i /ansible/hosts.ini /ansible/deploy.yml
                """
            }
        }
    }

    post {
        always {
            // Native Jenkins step to publish the HTML report. 
            publishHTML(allowMissing: true,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: '.',
                reportFiles: 'zap_report.html',
                reportName: 'ZAP Security Report',
                reportTitles: 'ZAP Baseline Scan Output'
            )
        }
    }
}
