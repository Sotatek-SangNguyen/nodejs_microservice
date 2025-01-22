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
                    dir(path: "${env.SERVICE_NAME}"){
                        sh 'echo $DOCKER_IMAGE_TAG'
                        sh 'npm install'
                        sh 'npm run test'
                    }
            }
        }
        stage('SonarQube analysis'){
            steps{
                dir("${env.SERVICE_NAME}"){
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
                dir("${env.SERVICE_NAME}"){
                    sh 'trivy fs . > trivy_scan_fs.txt'
                    sh 'cat trivy_scan_fs.txt'
                }
            }
        }
        stage('Build Docker image'){
            steps{
                dir(path: "${env.SERVICE_NAME}"){
                    script{
                        docker.build("${env.DOCKERHUB_REPO_NAME}:${env.DOCKER_IMAGE_TAG}")
                    }
                }
            }
        }
        stage('Scan Docker image'){
            steps{
                dir("${env.SERVICE_NAME}"){
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
        stage('Clean ws'){
            steps {
                cleanWs()
            }
        }
        stage('Checkout second time'){
            steps {
                script {
                    try {
                        def folderPath = 'manifest'
                        if (fileExists(folderPath)) {
                            sh "rm -rf ${folderPath}"
                        }
                        sh "mkdir -p ${folderPath}"
                        dir("${folderPath}") {
                            checkout([
                                $class: 'GitSCM',
                                branches: [[name: '*/main']],
                                userRemoteConfigs: [[
                                    url: 'https://github.com/Sotatek-SangNguyen/DevOps620-K8s-manifest.git',
                                    credentialsId: 'github-token'
                                ]]
                            ])
                        }
                        sh "ls ${folderPath}"
                    } catch(Exception e){
                        echo "Checkout failed: ${e.getMessage()}"
                    }
                    
                }
            }
        }
        stage('Update Deployment file') {
            environment {
                GIT_REPO_NAME = "DevOps620-K8s-manifest"
                GIT_USER_NAME = "Sotatek-SangNguyen"
            }
            steps {
            	dir("manifest/${env.SERVICE_NAME}"){
                    withCredentials([usernamePassword(credentialsId: 'github-token', usernameVariable: 'GITHUB_USER', passwordVariable: 'GITHUB_TOKEN')]) {
                        sh """
                            git config user.email "sang.nguyen@sotatek.com"
                            git config user.name "Nguyen Sang"
                            imageTag=\$(grep -oP '(?<=${env.SERVICE_NAME}:)[^ ]+' deployment.yaml)
                            echo \$imageTag
                            sed -i "s|${DOCKERHUB_REPO_NAME}:\$imageTag|${env.DOCKERHUB_REPO_NAME}:${env.DOCKER_IMAGE_TAG}|" deployment.yaml
                            git add deployment.yaml
                            git commit -m "Update deployment Image to version ${env.DOCKER_IMAGE_TAG}"
                            git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME}.git HEAD:main
                        """
                    }
                }
            }
        }
    }
}
