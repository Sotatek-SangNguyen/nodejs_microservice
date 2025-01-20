pipeline{
    agent any
    tools {
	nodejs 'NodeJS 20.18.1'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        // DOCKERHUB_REPO_NAME = 'registry.cloudemon.me/devops-620/customer'
        DOCKER_IMAGE_TAG = "${env.GIT_COMMIT[0..6]}"
        DOCKER_REGISTRY_URL = "registry.cloudemon.me"
    }
    stages{
        stage('Prepare'){
            steps {
                script {
                    def services = [
                        'customer': 'customer/',
                        'products': 'products/',
                        'shopping': 'shopping/'
                    ]

                    def changeLogSets = currentBuild.changeSets
                    def changedFiles = []
                    
                    
                    changeLogSets.each { changeSet ->
                        changeSet.getItems().each { item ->
                            item.getAffectedFiles().each { file ->
                                changedFiles.add(file.getPath())
                                println file.getPath()
                            }
                        }
                    }
                    
                    // Tìm service bị ảnh hưởng
                    def affectedService = null
                    
                    services.each { service, path ->
                        if (changedFiles.any { file -> file.startsWith(path) }) {
                            affectedService = service
                            echo "Found changes in service: ${service}"
                            echo "Changed files in ${service}:"
                            changedFiles.findAll { it.startsWith(path) }.each { file ->
                                echo "  - ${file}"
                            }
                        }
                    }
                    
                    if (affectedService) {
                        env.SERVICE_NAME = affectedService
                        env.DOCKERHUB_REPO_NAME = "registry.cloudemon.me/devops-620/${affectedService}"
                    } else {
                        error "No service changes detected"
                    }
                }
            }
        }
        stage('Unit test'){
            steps{		
                    dir(path: '${env.SERVICE_NAME}'){
                        sh 'echo $DOCKER_IMAGE_TAG'
                        sh 'npm install'
                        sh 'npm run test'
                    }
            }
        }
        stage('SonarQube analysis'){
            steps{
                dir('${env.SERVICE_NAME}'){
                    withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectKey=Devops-620 \
                        -Dsonar.sources=. \
                    '''
                }
                }
            }
        }

        stage('Scan filesystem'){
            steps{
                dir('${env.SERVICE_NAME}'){
                    sh 'trivy fs . > trivy_scan_fs.txt'
                    sh 'cat trivy_scan_fs.txt'
                }
            }
        }
        stage('Build Docker image'){
            steps{
                dir(path: '${env.SERVICE_NAME}'){
                    script{
                        docker.build("${env.DOCKERHUB_REPO_NAME}:${env.DOCKER_IMAGE_TAG}")
                    }
                }
            }
        }
        stage('Scan Docker image'){
            steps{
                dir('${env.SERVICE_NAME}'){
                    sh "trivy image ${env.DOCKERHUB_REPO_NAME}:${env.DOCKER_IMAGE_TAG} > trivy_scan_image.txt"
                    sh 'cat trivy_scan_image.txt'
                }
                
            }
        }
        stage("Docker Image Pushing") {
            steps {
            	withCredentials([usernamePassword(
                    credentialsId: 'harbor-token',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                        echo ${DOCKER_USER}
                        docker login ${DOCKER_REGISTRY_URL} -u \${DOCKER_USER} -p \${DOCKER_PASS}
                        docker push ${env.DOCKERHUB_REPO_NAME}:${env.DOCKER_IMAGE_TAG}
                         """
                    }
			}
        }
    }
}
