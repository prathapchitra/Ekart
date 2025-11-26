pipeline {

    agent {
        docker {
            image 'cprathap/maven-docker:latest'
            args '--user root -v /var/run/docker.sock:/var/run/docker.sock -v /var/lib/jenkins/.kube:/root/.kube'
        }
    }

    environment {
        KUBECONFIG = "/root/.kube/config"
        APP_NAME   = "ekart"
        IMAGE_NAME = "cprathap/ekart:latest"
    }

    stages {

        stage('Git Checkout') {
            steps {
                git branch: 'main', 'https://github.com/prathapchitra/Ekart.git'
            }
        }

        stage('Code Compile') {
            steps {
                sh 'mvn clean compile'
            }
        }

        stage('Unit Tests') {
            steps {
                sh 'mvn test'
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--scan ./', odcInstallation: 'DC'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh """
                        mvn sonar:sonar \
                        -Dsonar.projectKey=Ekart \
                        -Dsonar.projectName=Ekart \
                        -Dsonar.java.binaries=target
                    """
                }
            }
        }

        stage('Build Package') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker build -t ${IMAGE_NAME} ."
                    }
                }
            }
        }

        stage('Docker Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker push ${IMAGE_NAME}"
                    }
                }
            }
        }

        stage('Docker Image Scan (Trivy)') {
            steps {
                sh '''
                    export TRIVY_CACHE_DIR=/tmp/trivy-cache
                    mkdir -p $TRIVY_CACHE_DIR
                    trivy image --cache-dir $TRIVY_CACHE_DIR cprathap/ekart:latest
                '''
            }
        }

        stage('K8s Deployment') {
            steps {
                sh 'kubectl apply -f deployment-service.yaml'
            }
        }
    }

    post {
        success {
            echo "Pipeline SUCCESS: Ekart deployed successfully!"
        }
        failure {
            echo "Pipeline FAILED!"
        }
    }
}
