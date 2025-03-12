pipeline {
    agent {
        label "devops2402" 
    }

    tools {
        jdk "jdk17"
        maven "maven3"
        nodejs "nodejs23"
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {

        stage("Git Checkout") {
            steps {
                git(
                    branch: "${BRANCH_NAME}",
                    credentialsId: "git-credentials",
                    url: "https://github.com/VuPlayBoizz/Database-Service.git"
                )

                // Ghi commit-id vào file để dùng làm IMAGE_TAGS sau này
                sh "git rev-parse --short HEAD > commit-id"
            }
        }

        stage("Compile Code") {
            steps {
                sh "mvn clean compile"
            }
        }

        stage("Unit Test") {
            steps {
                sh "mvn test"
            }
        }

        stage("File System Scan") {
            steps {
                sh '''
                    trivy fs --format table -o trivy-fs-report.html .
                '''
            }            
        }

        stage("SonarQube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectName=Database-Service \
                        -Dsonar.projectKey=Database-Service \
                        -Dsonar.java.binaries=.
                    '''
                }
            }
        }

        stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, 
                    credentialsId: 'sonar-credentials'
                }
            }
        }

        stage("Docker Build Images") {
            steps {
                script {
                    // Đọc commit id từ file commit-id làm tag cho image Docker
                    env.IMAGE_TAGS = readFile('commit-id').trim()
                    env.IMAGE_NAME = "${DOCKER_USERNAME}/${PROJECT_NAME}:${IMAGE_TAGS}"

                    sh "docker build -t ${env.IMAGE_NAME} ."
                }
            }
        }
        
        stage("Setup AWS Credentials") {
            steps {
                withCredentials([aws(credentialsId: 'aws-credentials', region: "${REGION}")]) {  
                    script {
                        env.AWS_ACCESS_KEY_ID = env.AWS_ACCESS_KEY_ID
                        env.AWS_SECRET_ACCESS_KEY = env.AWS_SECRET_ACCESS_KEY
                    }
                    sh "aws sts get-caller-identity" 
                }
            }
        }

        stage("Create Private Repository on AWS ECR") {
            steps {
                script{
                    sh '''
                        echo "Checking if repository exists..."
                        aws ecr describe-repositories --repository-names ${PROJECT_NAME} --region ${REGION} || \
                        aws ecr create-repository --repository-name ${PROJECT_NAME} --region ${REGION}
                    '''
                }
            }
        }

        stage("Push Docker images to AWS ECR") {
            steps {
                script {
                    def ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com"
                    
                    echo "IMAGE_NAME: ${env.IMAGE_NAME}"
                    echo "ECR_REGISTRY: ${ECR_REGISTRY}"
                    echo "IMAGE_TAGS: ${env.IMAGE_TAGS}"

                    sh "aws ecr get-login-password --region ${REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}"
                    sh "docker tag ${env.IMAGE_NAME} ${ECR_REGISTRY}/${PROJECT_NAME}:${env.IMAGE_TAGS}"
                    sh "docker push ${ECR_REGISTRY}/${PROJECT_NAME}:${env.IMAGE_TAGS}"
                    sh "docker rmi -f ${env.IMAGE_NAME}"
                    sh "docker rmi -f ${ECR_REGISTRY}/${PROJECT_NAME}:${env.IMAGE_TAGS}"
                }
            }
        }

        stage("Load Database Credentials") {
            steps {
                withCredentials([file(credentialsId: 'database-credentials', variable: 'DATABASE_FILE')]) {
                    script {
                        sh "cp -f ${DATABASE_FILE} Deloyment/Database.Secret.yaml"
                    }
                }
            }
        }

        stage("Setup EKS Cluster Credentials") {
            steps {
                withKubeConfig(caCertificate: '', 
                clusterName: '', 
                contextName: '', 
                credentialsId: 'eks-credentials', 
                namespace: '', 
                restrictKubeConfigAccess: false, serverUrl: '') {
                    sh "kubectl get nodes"
                }                
            }
        }

        stage("Deploy to EKS Cluster") {
            steps {
                withKubeConfig(caCertificate: '', 
                clusterName: '', 
                contextName: '', 
                credentialsId: 'eks-credentials', 
                namespace: '', 
                restrictKubeConfigAccess: false, serverUrl: '') {
                    sh "kubectl create -f Deployment/ "
                }                
            }
        }
    } 
}
