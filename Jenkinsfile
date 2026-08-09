def notifySlack(String status, String emoji) {
    withCredentials([string(credentialsId: 'slack-webhook-url', variable: 'SLACK_WEBHOOK_URL')]) {
        sh """
            curl -s -X POST -H 'Content-type: application/json' \\
                --data '{"text":"${emoji} *${env.JOB_NAME}* #${env.BUILD_NUMBER} ${status}\\n${env.BUILD_URL}"}' \\
                "\$SLACK_WEBHOOK_URL"
        """
    }
}

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
        stage('Notify Start') {
            agent { label 'vps-host' }
            steps {
                script { notifySlack('started', ':large_blue_circle:') }
            }
        }

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
                    // BananaPi only has the legacy standalone docker-compose (v1.25.0),
                    // not the docker-compose-plugin ("docker compose") subcommand.
                    //
                    // --force-recreate is required, not optional: rm -rf public above
                    // deletes and recreates the directory (new inode) on every deploy,
                    // but a container's bind mount stays pinned to the original inode
                    // it was started with. Without recreating the container, its view
                    // of /usr/share/nginx/html goes stale/empty and nginx serves 403s.
                    sh 'docker-compose -f docker-compose.prod.yml up -d --force-recreate'
                }
            }
        }
    }

    post {
        success {
            node('vps-host') {
                script { notifySlack('succeeded', ':white_check_mark:') }
            }
        }
        failure {
            echo "Deploy failed — check docker compose logs on the BananaPi."
            node('vps-host') {
                script { notifySlack('failed', ':x:') }
            }
        }
    }
}
