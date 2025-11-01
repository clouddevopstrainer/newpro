pipeline {
    agent any

    environment {
        APP_NAME = "springboot-app"
        DOCKER_IMAGE = "preetz1303/${APP_NAME}"
        DOCKER_TAG = "v${BUILD_NUMBER}"
        MAVEN_HOME = tool 'Maven3'
        JAVA_HOME = tool 'JDK17'
        SONARQUBE_ENV = 'ec2-token'
        DOCKER_CRED = 'dockerhub-credentials'
        TERRAFORM_CRED  = 'aws-access-key'
        ANSIBLE_KEY = 'ec2-key-credentials-id'
        ANSIBLE_PLAYBOOK = 'ansible/setup.yml'
        K8S_MANIFEST = 'k8s/'  // optional Kubernetes manifest
        PATH = "${MAVEN_HOME}/bin:${JAVA_HOME}/bin:${env.PATH}"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/clouddevopstrainer/newpro.git'
            }
        }

        stage('Build & SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_ENV}") {
                    sh '''
                        mvn clean verify sonar:sonar \
                        -Dsonar.projectKey=cicd \
                        -Dsonar.projectName=cicd \
                        -Dsonar.java.binaries=target
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 20, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', "${DOCKER_CRED}") {
                        sh """
                            docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} .
                            docker tag ${DOCKER_IMAGE}:${BUILD_NUMBER} ${DOCKER_IMAGE}:latest
                            docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                            docker push ${DOCKER_IMAGE}:latest
                        """
                    }
                }
            }
        }

        stage('Terraform Apply Infra') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${TERRAFORM_CRED}",
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    dir('terraform') {
                        sh '''
                            terraform init -input=false
                            terraform plan -out=tfplan -input=false
                            terraform apply -auto-approve tfplan
                        '''
                    }
                }
            }
        }

        stage('Get EC2 Public IP') {
            steps {
                script {
                    env.EC2_PUBLIC_IP = sh(
                        script: 'terraform -chdir=terraform output -raw instance_public_ip',
                        returnStdout: true
                    ).trim()
                    echo "✅ EC2 Public IP: ${env.EC2_PUBLIC_IP}"
                }
            }
        }

        stage('Configure EC2 with Ansible') {
            steps {
                sshagent([ANSIBLE_KEY]) {
                    sh '''
                        chmod 600 ~/.ssh/id_rsa || true
                        echo "[ec2]" > temp_inventory.ini
                        echo "${EC2_PUBLIC_IP} ansible_user=ubuntu" >> temp_inventory.ini
                        export ANSIBLE_HOST_KEY_CHECKING=False
                        ansible-playbook -i temp_inventory.ini ${ANSIBLE_PLAYBOOK}
                    '''
                }
            }
        }

        stage('Optional: Deploy to Kubernetes') {
            steps {
                sshagent([ANSIBLE_KEY]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@${EC2_PUBLIC_IP} '
                        if [ -f /home/ubuntu/.kube/config ]; then
                            export KUBECONFIG=/home/ubuntu/.kube/config &&
                            kubectl apply -f ${K8S_MANIFEST} || echo "K8s manifests already applied"
                        else
                            echo "⚠️ No K8s config found, skipping."
                        fi
                        '
                    """
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                echo "✅ EC2 Public IP: ${env.EC2_PUBLIC_IP}"
                echo "Spring Boot App: http://${env.EC2_PUBLIC_IP}:8080/api/hello"
                echo "Prometheus: http://${env.EC2_PUBLIC_IP}:9090"
                echo "Grafana: http://${env.EC2_PUBLIC_IP}:3000 (user: admin / password: admin)"
            }
        }
    }

    post {
        success {
            mail to: 'preethidora03@gmail.com',
                subject: "✅ SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Good news! The Jenkins build succeeded.\n\nJob: ${env.JOB_NAME}\nBuild URL: ${env.BUILD_URL}"
        }
        failure {
            mail to: 'preethidora03@gmail.com',
                subject: "❌ FAILURE: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "The Jenkins build failed.\n\nPlease check: ${env.BUILD_URL}"
        }
    }
}
