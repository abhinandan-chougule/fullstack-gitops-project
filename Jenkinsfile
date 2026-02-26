@Library('Shared') _
pipeline {
    agent { label 'Node' }
    
    environment {
        SONAR_HOME = tool "Sonar"
        DC_DATA_DIR = "/var/lib/dependency-check"   // persistent cache
    }
    
    parameters {
        string(name: 'FRONTEND_DOCKER_TAG', defaultValue: '', description: 'Setting docker image for latest push')
        string(name: 'BACKEND_DOCKER_TAG', defaultValue: '', description: 'Setting docker image for latest push')
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

        stage("Workspace cleanup") {
            steps {
                cleanWs()
            }
        }
        
        stage('Git: Code Checkout') {
            steps {
                script {
                    code_checkout(
                        "https://github.com/abhinandan-chougule/fullstack-gitops-project.git",
                        "main"
                    )
                }
            }
        }
        
        stage("Trivy: Filesystem scan") {
            steps {
                script {
                    trivy_scan()
                }
            }
        }

        stage("OWASP: Dependency check") {
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
                    withSonarQubeEnv('http://18.139.84.52:9000') {
                        sonarqube_analysis("Sonar", "fullstack", "fullstack")
                    }
                }
            }
        }
        
        stage("SonarQube: Code Quality Gates") {
            steps {
                script {
                    timeout(time: 5, unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: true
                    }
                }
            }
        }
        
        stage('Exporting environment variables') {
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
                    dir('backend') {
                        docker_build("fullstack-backend-beta", "${params.BACKEND_DOCKER_TAG}", "abhic25")
                    }
                    
                    dir('frontend') {
                        docker_build("fullstack-frontend-beta", "${params.FRONTEND_DOCKER_TAG}", "abhic25")
                    }
                }
            }
        }
        
        stage("Docker: Push to DockerHub") {
            steps {
                script {
                    docker_push("fullstack-backend-beta", "${params.BACKEND_DOCKER_TAG}", "abhic25") 
                    docker_push("fullstack-frontend-beta", "${params.FRONTEND_DOCKER_TAG}", "abhic25")
                }
            }
        }
    }

    post {
        success {
            archiveArtifacts artifacts: '*.xml', followSymlinks: false
            
            build job: "Wanderlust-CD", parameters: [
                string(name: 'FRONTEND_DOCKER_TAG', value: "${params.FRONTEND_DOCKER_TAG}"),
                string(name: 'BACKEND_DOCKER_TAG', value: "${params.BACKEND_DOCKER_TAG}")
            ]
        }
    }
}
