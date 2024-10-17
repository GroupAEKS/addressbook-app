pipeline {
    agent { node { label "maven-sonarqube-node" } }

    parameters {
        choice(name: 'Environment', choices: ['Dev', 'QA', 'UAT', 'Prod'], description: 'Target environment for deployment')
        string(name: 'ecr_tag', defaultValue: '1.6.0', description: 'Assign the ECR tag version for the build')
    }

    tools {
        maven "maven3.9.8"
    }

    stages {
        stage('1. Git Checkout') {
            steps {
                git branch: 'release', credentialsId: 'Github-pat', url: 'https://github.com/GroupAEKS/addressbook-app'
            }
        }

        stage('2. Build with Maven') {
            steps {
                sh "mvn clean package"
            }
        }

        stage('3. SonarQube Analysis') {
            environment {
                scannerHome = tool 'SonarQube-Scanner-6.2.1'
            }
            steps {
                withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
                    sh """
                    ${scannerHome}/bin/sonar-scanner  \
                    -Dsonar.projectKey=addressbook-application \
                    -Dsonar.projectName='addressbook-application' \
                    -Dsonar.host.url=http://18.246.39.248:9000 \
                    -Dsonar.login=${SONAR_TOKEN} \
                    -Dsonar.sources=src/main/java/ \
                    -Dsonar.java.binaries=target/classes
                    """
                }
            }
        }
      stage('4. Docker Image Build') {
        steps {
          sh "aws ecr get-login-password --region us-west-2 | sudo docker login --username AWS --password-stdin ${aws_account}.dkr.ecr.us-west-2.amazonaws.com"
          sh "sudo docker build -t addressbook ."
          sh "sudo docker tag addressbook:latest ${aws_account}.dkr.ecr.us-west-2.amazonaws.com/addressbook:${params.ecr_tag}"
          sh "sudo docker push ${aws_account}.dkr.ecr.us-west-2.amazonaws.com/addressbook:${params.ecr_tag}"
        }
    }

        
        stage('5. Application Deployment in EKS') {
            steps {
                kubeconfig(caCertificate: '', credentialsId: 'kubeconfig', serverUrl: '') {
                    sh "kubectl apply -f manifest"
                }
            }
        }

        stage('6. Monitoring Solution Deployment in EKS') {
            steps {
                kubeconfig(caCertificate: '', credentialsId: 'kubeconfig', serverUrl: '') {
                    sh "kubectl apply -k monitoring"
                    sh "script/install_helm.sh"
                    sh "script/install_prometheus.sh"
                }
            }
        }

        stage('7. Email Notification') {
            steps {
                mail bcc: 'pamyleitich@gmail.com', 
                     body: '''Build is complete. Check the application using the URL below:
https://app.dominionsystem.org/addressbook-1.0
Let me know if the changes look okay.

Thanks,
Dominion System Technologies,
+1 (313) 413-1477''', 
                     subject: 'Application Successfully Deployed!', 
                     to: 'pamyleitich@gmail.com'
            }
        }
    }
}
