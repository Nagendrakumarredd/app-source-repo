pipeline {
    agent any
    environment {
        DOCKER_IMAGE = 'bhanutejaravutla/simple-app'
    }
    stages {
        stage('Maven Compile & Build') {
            steps { sh 'mvn clean package -DskipTests' }
        }
        stage('Bypass Scans for PoC') {
            steps { echo "Bypassing scan step to clear git delivery pipeline." }
        }
        stage('Package & Push Container') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-creds', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                    sh """
                        echo "\$DOCKER_PASSWORD" | docker login -u "\$DOCKER_USERNAME" --password-stdin
                        docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} .
                        docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                        docker rmi ${DOCKER_IMAGE}:${BUILD_NUMBER} || true
                    """
                }
            }
        }
        stage('Manifest GitOps Delivery Loop') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'git-creds', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                        sh """
                            git config --global user.email "jenkins-bot@poc.com"
                            git config --global user.name "Jenkins GitOps Engine"
                            rm -rf target-manifests
                            
                            # Clean, explicit, hardcoded clone command string
                            git clone https://\$GIT_USERNAME:\$GIT_PASSWORD@://github.com target-manifests
                            
                            cd target-manifests
                            sed -i "s|image: bhanutejaravutla/simple-app:.*|image: bhanutejaravutla/simple-app:${BUILD_NUMBER}|g" deployment.yaml
                            git add deployment.yaml
                            git commit -m "chore: auto-bump tag to release-${BUILD_NUMBER} [skip ci]" || echo "No changes to commit"
                            git push origin main
                        """
                    }
                }
            }
        }
    }
}
