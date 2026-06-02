pipeline {
    agent any
    environment {
        DOCKER_IMAGE     = 'syamalanagendrakumarreddy/simple-app'
        SONAR_ORG        = 'nagendrakumarredd'
        SONAR_PROJ       = 'Nagendrakumarredd_app-source-repo'
        JAVA_TOOL_OPTIONS = "-Xms512m -Xmx1024m -XX:MaxMetaspaceSize=512m"
    }

    stages {
        stage('Maven Compile & Build') {
            steps { sh 'mvn clean package -DskipTests' }
        }
        
         stage('SonarCloud Scan Validation') {
            steps {
                withSonarQubeEnv('SonarCloud') { 
                    withCredentials([string(credentialsId: 'sonarcloud-token', variable: 'SONAR_TOKEN')]) {
                        
                        sh """
                        export MAVEN_OPTS="-Xms512m -Xmx1024m -XX:MaxMetaspaceSize=512m"
                        
                        mvn sonar:sonar \
                        -Dsonar.host.url=https://sonarcloud.io \
                        -Dsonar.token=\$SONAR_TOKEN \
                        -Dsonar.organization=${SONAR_ORG} \
                        -Dsonar.projectKey=${SONAR_PROJ} \
                        -Dsonar.workers=1 \
                        -Dsonar.scanAllFiles=false \
                        -Dsonar.java.binaries=target/classes
                        """
                    }
                }
            }
        }

        
        stage('Verify Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    script {
                        def qg = waitForQualityGate()
                        if ( qg.status != 'NONE' && qg.status !='OK') { 
                            error "Pipeline stopped: Quality Gate failed: ${qg.status}" 
                        } else {
                            echo "Quality Gate check cleared with status: ${qg.status}"
                        }
                    }
                }
            }
        }

        stage('Execute Unit Tests') {
            steps { sh 'mvn test' }
        }
        
        stage('Package & Push Container') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-hub-creds',
                    usernameVariable: 'DOCKER_USERNAME',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {
        
                    sh """
                    echo "Docker Image: ${DOCKER_IMAGE}"
                    echo "Build Number: ${BUILD_NUMBER}"
        
                    echo "\$DOCKER_PASSWORD" | docker login -u "\$DOCKER_USERNAME" --password-stdin
        
                    docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} .
        
                    docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
        
                    docker rmi ${DOCKER_IMAGE}:${BUILD_NUMBER} || true
                    """
                }
            }
        }

stage('Update Manifest Repo (GitOps)') {
    steps {
        script {
            // Jenkins natively handles mask / hide for these credentials
            withCredentials([usernamePassword(
                credentialsId: 'git-pat',
                usernameVariable: 'GIT_USER',
                passwordVariable: 'GIT_TOKEN'
            )]) {

                sh '''
                # 1. Setup identity
                git config --global user.email "jenkins-bot@poc.com"
                git config --global user.name "Jenkins GitOps Engine"

                # 2. Fresh clone
                rm -rf target-manifests
                git clone https://github.com target-manifests
                cd target-manifests

                # 3. Update build number in manifest
                sed -i "s|image:.*|image: $DOCKER_IMAGE:$BUILD_NUMBER|g" deployment.yaml

                # 4. Commit changes safely
                git add .
                git commit -m "Update image to $BUILD_NUMBER" || echo "No changes to commit"

                # 5. Native authenticated push (No headers or base64 required)
                git push https://${GIT_USER}:${GIT_TOKEN}@://github.com main
                '''
            }
        }
    }
}


    }
}
