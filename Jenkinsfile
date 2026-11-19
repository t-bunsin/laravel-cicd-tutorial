pipeline {
    agent any
    environment {
        // No server yet: set to 'true' once the aws-ec2 SSH credential and a server exist
        DEPLOY_ENABLED = 'false'
        DEPLOY_HOST = 'ec2-user@13.40.116.143'
    }
    stages {
        stage("Verify tooling") {
            steps {
                sh '''
                    docker info
                    docker version
                    docker compose version
                '''
            }
        }
        stage("Verify SSH connection to server") {
            when { environment name: 'DEPLOY_ENABLED', value: 'true' }
            steps {
                sshagent(credentials: ['aws-ec2']) {
                    sh 'ssh -o StrictHostKeyChecking=no $DEPLOY_HOST whoami'
                }
            }
        }
        stage("Clear this project's docker containers") {
            steps {
                // docker-compose.yml uses fixed container_name values, so remove containers
                // with those names (including a local dev stack) — but nothing else
                sh 'docker rm -f nginx php mysql redis mailhog composer npm artisan || true'
            }
        }
        stage("Start Docker") {
            steps {
                sh 'make up'
                sh 'docker compose ps'
            }
        }
        stage("Run Composer Install") {
            steps {
                sh 'docker compose run --rm composer install'
            }
        }
        stage("Populate .env file") {
            steps {
                // Keep the real .env outside git, in $JENKINS_HOME/workspace/envs/laravel-test/.env
                sh 'cp "$JENKINS_HOME/workspace/envs/laravel-test/.env" "$WORKSPACE/.env"'
            }
        }
        stage("Run Tests") {
            steps {
                sh 'docker compose run --rm artisan test'
            }
        }
    }
    post {
        success {
            sh 'rm -rf artifact.zip'
            sh 'zip -qr artifact.zip . -x "*node_modules**" -x ".git/*"'
            // Downloadable from the build page in Jenkins
            archiveArtifacts artifacts: 'artifact.zip', fingerprint: true
            script {
                if (env.DEPLOY_ENABLED == 'true') {
                    withCredentials([sshUserPrivateKey(credentialsId: "aws-ec2", keyFileVariable: 'keyfile')]) {
                        sh 'scp -o StrictHostKeyChecking=no -i ${keyfile} "$WORKSPACE/artifact.zip" $DEPLOY_HOST:/home/ec2-user/artifact'
                    }
                    sshagent(credentials: ['aws-ec2']) {
                        sh 'ssh -o StrictHostKeyChecking=no $DEPLOY_HOST unzip -o /home/ec2-user/artifact/artifact.zip -d /var/www/html'
                        try {
                            sh 'ssh -o StrictHostKeyChecking=no $DEPLOY_HOST sudo chmod 777 /var/www/html/storage -R'
                        } catch (Exception e) {
                            echo 'Some file permissions could not be updated.'
                        }
                    }
                } else {
                    echo 'DEPLOY_ENABLED is false: skipping deploy to server.'
                }
            }
        }
        always {
            sh 'docker compose down --remove-orphans -v'
            sh 'docker compose ps'
        }
    }
}
