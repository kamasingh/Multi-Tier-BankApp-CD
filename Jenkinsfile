pipeline {
    agent any
    
    parameters {
        string(name:'DOCKER_TAG' , defaultValue: 'latest' , description: 'Docker tag')
    }
    
    tools {
        maven 'maven3'
    }
    
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        IMAGE_TAG = "v${BUILD_NUMBER}"
    }

    stages {
        stage('Git Checkout') {
            steps {
               git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/kamasingh/Multi-Tier-BankApp-CI.git'
            }
        }
        
        stage('compile') {
            steps {
                sh "mvn compile"
            }
        }
        
        stage('test') {
            steps {
                sh "mvn test -DskipTests=true"
            }
        }
        
        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs --format table -o fs-report.html .'
                archiveArtifacts 'fs-report.html'
            }
        }
        
        stage('Sonar Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectKey=bankapp \
                        -Dsonar.projectName=bankapp \
                        -Dsonar.java.binaries=target
                    '''
                }
            }
        }
     
        
        stage('Publish To Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'maven-settings', maven: 'maven3', traceability: true) {
                    sh "mvn deploy -DskipTests=true"
                }
            }
        }
        
        stage('Docker Image Build & Tag') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-creds') {
                        sh "docker build -t kama1234/bankapp:${params.DOCKER_TAG} ."
                    }
                }
            }
        }
        
        stage('Docker Image') {
            steps {
                sh "trivy image --format table -o dimage.html adijaiswal/bankapp:${params.DOCKER_TAG}"
            }
        }
        
        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-creds') {
                        sh "docker push kama1234/bankapp:${params.DOCKER_TAG}"
                    }
                }
            }
        }
        
        
          // Update the image tag in the YAML manifest in another repository 
        stage('Update YAML Manifest in Other Repo') {
    steps {
        script {
           withCredentials([gitUsernamePassword(credentialsId: 'git-cred', gitToolName: 'Default')]) {
                sh '''
                    # Remove existing directory if it exists
                    rm -rf Multi-Tier-BankApp-CD

                    # Clone the repo
                    git clone https://github.com/kamasingh/Multi-Tier-BankApp-CD.git
                    cd Multi-Tier-BankApp-CD

                    # Verify file exists
                    if [ ! -f bankapp/bankapp-ds.yml ]; then
                        echo "ERROR: bankapp-ds.yml not found"
                        exit 1
                    fi

                    # Update Docker image tag
                    sed -i "s|image: kama1234/bankapp:.*|image: kama1234/bankapp:${DOCKER_TAG}|" bankapp/bankapp-ds.yml

                    # Show updated file
                    echo "Updated YAML file contents:"
                    cat bankapp/bankapp-ds.yml

                    # Configure Git
                    git config user.email "office@devopsshack.com"
                    git config user.name "DevOps Shack"

                    # Commit and push
                    git add bankapp/bankapp-ds.yml
                    git commit -m "Update image tag to ${DOCKER_TAG}"
                    git push origin main
                '''
            }
        }
    }

    }
    }

}
