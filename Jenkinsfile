pipeline {
    agent { node { label "maven-sonarqube-node" } }

    parameters {
        choice(name: 'aws_account', choices: ['999568710647', '4568366404742', '922266408974', '576900672829'], description: 'AWS account hosting image registry')
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

        stage('4. Docker Build and Push to Public ECR') {
            environment {
                AWS_REGION = 'us-east-1' // Set the region for public ECR
                ECR_PUBLIC_REPO = 'public.ecr.aws/e4o4k3j4/team1' // Replace with your ECR Public repo URL
            }

            steps {
                script {
                    // Authenticate with ECR Public
                    sh '''
                        echo "Authenticating with AWS ECR Public..."
                        aws ecr-public get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_PUBLIC_REPO}
                    '''
                    
                    // Build the Docker image
                    sh '''
                        echo "Building Docker image..."
                        docker build -t team1 .
                    '''
                    
                    // Tag the Docker image
                    sh '''
                        echo "Tagging Docker image..."
                        docker tag team1:latest ${ECR_PUBLIC_REPO}:${params.ecr_tag}
                    '''
                    
                    // Push the Docker image to Public ECR
                    sh '''
                        echo "Pushing Docker image to Public ECR..."
                        docker push ${ECR_PUBLIC_REPO}:${params.ecr_tag}
                    '''
                }
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
