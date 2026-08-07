pipeline {
    agent { label 'bananapi' }

    options {
        disableConcurrentBuilds()
        timeout(time: 10, unit: 'MINUTES')
    }

    environment {
        REPO_DIR = '/home/jenkins/repos/blog'
    }

    stages {
        stage('Pull') {
            steps {
                dir("${REPO_DIR}") {
                    sh 'git pull origin main'
                }
            }
        }

        stage('Build') {
            steps {
                dir("${REPO_DIR}") {
                    sh '''
                        docker run --rm -u "$(id -u):$(id -g)" -v "$PWD:/app" --workdir /app \
                            ghcr.io/getzola/zola:v0.22.1 build
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                dir("${REPO_DIR}") {
                    sh 'docker compose -f docker-compose.prod.yml up -d'
                }
            }
        }
    }

    post {
        failure {
            echo "Deploy failed — check docker compose logs on the BananaPi."
        }
    }
}
