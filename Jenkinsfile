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

stage('Manifest GitOps Delivery Loop') {
    steps {
        script {
            withCredentials([usernamePassword(credentialsId: 'git-pat', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
                sh '''
                git config --global user.email "jenkins-bot@poc.com"
                git config --global user.name "Jenkins GitOps Engine"

                rm -rf target-manifests

                # Encode the password/PAT to bypass any hyphen or parsing errors entirely
                ENCODED_TOKEN=$(echo -n "${GIT_USER}:${GIT_TOKEN}" | base64 | tr -d '\n')

                # Clone using an inline HTTP authorization header string
                git -c http.extraHeader="Authorization: Basic ${ENCODED_TOKEN}" clone https://github.com target-manifests

                cd target-manifests

                # Configure the local repository to use the exact same authorization header automatically for pushing
                git config http.extraHeader "Authorization: Basic ${ENCODED_TOKEN}"

                # Update the target image deployment manifest file
                sed -i "s|image: .*|image: ${DOCKER_IMAGE}:${BUILD_NUMBER}|" deployment.yaml

                git add .
                git commit -m "Update image to build ${BUILD_NUMBER}" || echo "No changes"

                git push origin main
                '''
            }
        }
    }
}


    }
}
