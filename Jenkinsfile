@Library('Shared') _
pipeline {
    agent { label 'Node' }
    
    environment {
        SONAR_HOME = tool "Sonar"
        DC_DATA_DIR = "/var/lib/dependency-check"
    }
    
    parameters {
        string(name: 'FRONTEND_DOCKER_TAG', defaultValue: 'frontend', description: 'Base frontend docker tag')
        string(name: 'BACKEND_DOCKER_TAG', defaultValue: 'backend', description: 'Base backend docker tag')
    }
    
    stages {
        stage("Validate Parameters") {
            steps {
                script {
                    if (params.FRONTEND_DOCKER_TAG == '' || params.BACKEND_DOCKER_TAG == '') {
                        error("FRONTEND_DOCKER_TAG and BACKEND_DOCKER_TAG must be provided.")
                    }
                }
            }
        }

        stage("Workspace Cleanup") {
            steps {
                cleanWs()
            }
        }
        
        stage("Git: Code Checkout") {
            steps {
                script {
                    code_checkout(
                        "https://github.com/abhinandan-chougule/fullstack-gitops-project.git",
                        "main"
                    )
                }
            }
        }
        
        stage("Trivy: Filesystem Scan") {
            steps {
                script {
                    trivy_scan()
                }
            }
        }

        stage("OWASP: Dependency Check") {
            steps {
                withCredentials([string(credentialsId: 'NVD_API_KEY', variable: 'NVD_API_KEY')]) {
                    script {
                        owasp_dependency(
                            "--nvdApiKey=${NVD_API_KEY} --data ${DC_DATA_DIR}"
                        )
                    }
                }
            }
        }
        
        stage("SonarQube: Code Analysis") {
            steps {
                script {
                    sonarqube_analysis("Sonar", "fullstack", "fullstack")
                }
            }
        }
        
        stage("SonarQube: Quality Gate") {
            steps {
                script {
                    timeout(time: 10, unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: true
                    }
                }
            }
        }
        
        stage("Exporting Environment Variables") {
            parallel {
                stage("Backend env setup") {
                    steps {
                        dir("Automations") {
                            sh "bash updatebackendnew.sh"
                        }
                    }
                }
                
                stage("Frontend env setup") {
                    steps {
                        dir("Automations") {
                            sh "bash updatefrontendnew.sh"
                        }
                    }
                }
            }
        }
        
        stage("Docker: Build Images") {
            steps {
                script {
                    // Generate unique tags per build
                    def backendTag = "${params.BACKEND_DOCKER_TAG}-${env.BUILD_NUMBER}"
                    def frontendTag = "${params.FRONTEND_DOCKER_TAG}-${env.BUILD_NUMBER}"

                    dir('backend') {
                        docker_build("fullstack-backend-beta", backendTag, "abhic25")
                    }
                    dir('frontend') {
                        docker_build("fullstack-frontend-beta", frontendTag, "abhic25")
                    }

                    // Update manifests with new tags
                    dir('kubernetes') {
                        sh """
                            sed -i 's|fullstack-backend-beta:.*|fullstack-backend-beta:${backendTag}|g' backend.yaml
                            sed -i 's|fullstack-frontend-beta:.*|fullstack-frontend-beta:${frontendTag}|g' frontend.yaml
                        """
                    }
                }
            }
        }
        
        stage("Docker: Push to DockerHub") {
            steps {
                script {
                    def backendTag = "${params.BACKEND_DOCKER_TAG}-${env.BUILD_NUMBER}"
                    def frontendTag = "${params.FRONTEND_DOCKER_TAG}-${env.BUILD_NUMBER}"

                    docker_push("fullstack-backend-beta", backendTag, "abhic25") 
                    docker_push("fullstack-frontend-beta", frontendTag, "abhic25")
                }
            }
        }

        stage("Git: Commit and Push Manifests") {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'Github-cred',
                        usernameVariable: 'GIT_USER',
                        passwordVariable: 'GIT_TOKEN'
                    )
                ]) {
                    sh '''
                        git config user.name "Jenkins"
                        git config user.email "jenkins@ci.com"
                        git add kubernetes/*.yaml
                        git diff --cached --quiet || git commit -m "Updated image tags for build ${BUILD_NUMBER}"
                        git remote set-url origin https://$GIT_USER:$GIT_TOKEN@github.com/abhinandan-chougule/fullstack-gitops-project.git
                        git push origin main
                    '''
                }
            }
        }
    }

    post {
        success {
            archiveArtifacts artifacts: '*.xml', followSymlinks: false
            
            build job: "fullstack-ci", parameters: [
                string(name: 'FRONTEND_DOCKER_TAG', value: "${params.FRONTEND_DOCKER_TAG}-${env.BUILD_NUMBER}"),
                string(name: 'BACKEND_DOCKER_TAG', value: "${params.BACKEND_DOCKER_TAG}-${env.BUILD_NUMBER}")
            ]
        }
        failure {
            echo "Pipeline failed. Please check the logs."
        }
    }
}
