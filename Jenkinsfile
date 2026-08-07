pipeline {
    agent none

    options {
        disableConcurrentBuilds()
        timeout(time: 10, unit: 'MINUTES')
    }

    environment {
        REPO_DIR = '/home/jenkins/repos/blog'
    }

    stages {
        // ghcr.io/getzola/zola only publishes amd64/arm64 images — BananaPi is 32-bit
        // armv7, so the build has to happen on vps-host (amd64) and the result gets
        // handed to bananapi for deploy.
        stage('Build') {
            agent { label 'vps-host' }
            steps {
                checkout scm
                sh '''
                    docker run --rm -u "$(id -u):$(id -g)" -v "$PWD:/app" --workdir /app \
                        ghcr.io/getzola/zola:v0.22.1 build
                '''
                stash name: 'site', includes: 'public/**'
            }
        }

        stage('Deploy') {
            agent { label 'bananapi' }
            steps {
                dir("${REPO_DIR}") {
                    sh 'git pull origin main'
                    sh 'rm -rf public'
                    unstash 'site'
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
