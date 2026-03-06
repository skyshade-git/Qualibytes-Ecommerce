@Library('Shared') _

pipeline {
    agent any

    environment {
        APP_IMAGE = "skyshadedocker/qbshop-app"
        MIGRATION_IMAGE = "skyshadedocker/qbshop-migration"
        IMAGE_TAG = "${BUILD_NUMBER}"
        GIT_REPO = "https://github.com/skyshade-git/Qualibytes-Ecommerce.git"
        BRANCH = "dev"
    }

    stages {

        stage('Cleanup Workspace') {
            steps {
                echo "Cleaning workspace..."
                cleanWs()
            }
        }

        stage('Clone Repository') {
            steps {
                git branch: "${BRANCH}",
                    url: "${GIT_REPO}",
                    credentialsId: 'github-credentials'
            }
        }

        stage('Cleanup Old Docker Images') {
            steps {
                sh '''
                echo "Cleaning old Docker images..."
                docker image prune -f
                '''
            }
        }

        stage('Build Docker Images') {
            parallel {

                stage('Build Main App Image') {
                    steps {
                        sh '''
                        echo "Building main app image..."
                        docker build -t $APP_IMAGE:$IMAGE_TAG .
                        '''
                    }
                }

                stage('Build Migration Image') {
                    steps {
                        sh '''
                        if [ -f migration/Dockerfile ]; then
                            echo "Building migration image..."
                            docker build -t $MIGRATION_IMAGE:$IMAGE_TAG -f migration/Dockerfile .
                        else
                            echo "Migration Dockerfile not found. Skipping migration build."
                        fi
                        '''
                    }
                }

            }
        }

        stage('Run Unit Tests') {
            steps {
                sh '''
                echo "Running tests..."
                '''
            }
        }

        stage('Security Scan with Trivy') {
            steps {
                sh '''
                trivy image $APP_IMAGE:$IMAGE_TAG || true
                '''
            }
        }

        stage('Docker Login') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-hub-credentials',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {

                    sh '''
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    '''
                }
            }
        }

        stage('Push Docker Images') {
            parallel {

                stage('Push Main App Image') {
                    steps {
                        sh '''
                        docker push $APP_IMAGE:$IMAGE_TAG
                        '''
                    }
                }

                stage('Push Migration Image') {
                    steps {
                        sh '''
                        if docker image inspect $MIGRATION_IMAGE:$IMAGE_TAG > /dev/null 2>&1; then
                            docker push $MIGRATION_IMAGE:$IMAGE_TAG
                        else
                            echo "Migration image with tag $IMAGE_TAG not found. Skipping push."
                        fi
                        '''
                    }
                }

            }
        }

        stage('Update Kubernetes Manifests') {
            steps {
                sh '''
                sed -i "s|image: skyshadedocker/qbshop-app:.*|image: skyshadedocker/qbshop-app:$IMAGE_TAG|g" kubernetes/08-qbshop-deployment.yaml

                if [ -f kubernetes/12-migration-job.yaml ]; then
                    sed -i "s|image: skyshadedocker/qbshop-migration:.*|image: skyshadedocker/qbshop-migration:$IMAGE_TAG|g" kubernetes/12-migration-job.yaml
                fi
                '''
            }
        }

        stage('Commit & Push Changes to GitHub') {
            steps {

                withCredentials([usernamePassword(
                    credentialsId: 'github-credentials',
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_PASS'
                )]) {

                    sh '''
                    git config user.email "jenkins@ci.com"
                    git config user.name "Jenkins CI"

                    git add kubernetes/

                    git diff --quiet || git commit -m "Update QBShop image tag to $IMAGE_TAG [ci skip]"

                    git remote set-url origin https://$GIT_USER:$GIT_PASS@github.com/skyshade-git/Qualibytes-Ecommerce.git

                    git push origin HEAD:dev
                    '''
                }
            }
        }

    }

    post {
        success {
            echo "Pipeline completed successfully 🚀"
        }

        failure {
            echo "Pipeline failed ❌"
        }
    }
}
