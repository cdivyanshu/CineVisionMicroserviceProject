pipeline {
    agent any

    environment {
    }

    parameters {
        choice(name: 'BRANCH', choices: ['main', 'dev', 'feature1', 'UAT'], description: 'Select the branch to build and deploy')
        choice(name: 'MODULE', choices: ['ALL', 'frontend', 'api-gateway', 'emailService', 'eureka-server', 'movieService', 'userService'], description: 'Select the module to build and deploy')
    }

    stages {

        stage("Checkout Code") {
            steps {
                script {
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: "*/${params.BRANCH}"]],
                        userRemoteConfigs: [[
                            url: "${env.REPO_URL}",
                            credentialsId: "${env.GIT_CREDENTIALS_ID}"
                        ]]
                    ])
                    echo "Code checked out successfully from branch '${params.BRANCH}'."
                }
            }
        }

        stage('Determine Modules to Build') {
            steps {
                script {
                    if (params.MODULE == 'ALL') {
                        env.CHANGED_MODULES = 'frontend,api-gateway,emailService,eureka-server,movieService,userService'
                        echo "Building all modules: ${env.CHANGED_MODULES}"
                    } else {
                        env.CHANGED_MODULES = params.MODULE
                        echo "Building selected module: ${env.CHANGED_MODULES}"
                    }
                }
            }
        }

        stage('Build Modules') {
            environment {
                JAVA_HOME = "/usr/lib/jvm/java-21-openjdk-amd64"
                PATH = "${JAVA_HOME}/bin:${env.PATH}"
            }
            steps {
                script {
                    def modules = env.CHANGED_MODULES.split(',')

                    echo "Building parent POM for Java modules..."
                    sh 'mvn clean install -DskipTests'

                    modules.each { module ->
                        def trimmedModule = module.trim()

                        if (trimmedModule == 'frontend') {
                            echo "Building React Application for module: ${trimmedModule}"
                            dir("${trimmedModule}") {
                                sh """
                                    npm install
                                    npm run build
                                """
                            }
                        } else {
                            echo "Building Maven project for module: ${trimmedModule}"
                            dir("${trimmedModule}") {
                                sh 'mvn clean package -DskipTests'
                            }
                        }
                    }
                }
            }
        }

        stage('Verify Build Output') {
            steps {
                script {
                    def modules = env.CHANGED_MODULES.split(',')

                    modules.each { module ->
                        dir("${module.trim()}") {
                            sh """
                                echo "Verifying build output for module: ${module}"
                                ls -la || true
                                ls -la target/ || true
                                ls -la target/classes/ || true
                                find . -name "common-properties" || true
                            """
                        }
                    }
                }
            }
        }

        /*stage('Build and Push Docker Images') {
            steps {
                script {
                    def modules = env.CHANGED_MODULES.split(',')

                    withCredentials([usernamePassword(
                        credentialsId: env.Harbor_cred,
                        usernameVariable: 'HARBOR_USER',
                        passwordVariable: 'HARBOR_PASS'
                        )]) {

                        sh """
                            docker login ${env.HARBOR_REGISTRY} -u ${HARBOR_USER} -p ${HARBOR_PASS}
                        """

                        modules.each { module ->
                            def tag = "${env.BRANCH}-${env.BUILD_NUMBER}"

                            dir("${module.trim()}") {
                                echo "Building Docker image for module: ${module} with tag: ${tag}"
                                sh """
                                    docker build -t ${env.IMAGE_BASE_PATH}/${module}:${tag} .
                                    docker push ${env.IMAGE_BASE_PATH}/${module}:${tag}
                                """
                            }
                        }

                        sh 'docker logout ${HARBOR_REGISTRY}'
                    }
                }
            }
        }*/
        stage('Build and Push Docker Images') {
            steps {
                script {
                    def modules = env.CHANGED_MODULES.split(',')

                    withCredentials([usernamePassword(
                        credentialsId: env.Harbor_cred,
                        usernameVariable: 'HARBOR_USER',
                        passwordVariable: 'HARBOR_PASS'
                    )]) {

                        sh """
                            docker login ${env.HARBOR_REGISTRY} -u ${HARBOR_USER} -p ${HARBOR_PASS}
                        """

                        modules.each { module ->
                            def trimmedModule = module.trim()
                            def lowerModule = trimmedModule.toLowerCase() // convert to lowercase
                            def tag = "${env.BRANCH}-${env.BUILD_NUMBER}"

                            dir("${trimmedModule}") {
                                echo "Building Docker image for module: ${trimmedModule} with tag: ${tag}"

                                sh """
                                    docker build -t ${env.IMAGE_BASE_PATH}/${lowerModule}:${tag} .
                                    docker push ${env.IMAGE_BASE_PATH}/${lowerModule}:${tag}
                                """
                            }
                        }

                        sh "docker logout ${env.HARBOR_REGISTRY}"
                    }
                }
            }
        }


        stage('Deploy to Kubernetes') {
            steps {
                script {
                    def modules = env.CHANGED_MODULES.split(',')

                    modules.each { module ->
                        def deploymentFile = "k8s/${module.trim()}/deployment.yaml"
                        def serviceFile = "k8s/${module.trim()}/service.yaml"

                        echo "Deploying module: ${module.trim()} to Kubernetes"

                // Check if both files exist before applying
                        if (fileExists(deploymentFile) && fileExists(serviceFile)) {
                            sh """
                                kubectl apply -f ${deploymentFile}
                                kubectl apply -f ${serviceFile}
                            """
                        } else {
                            echo "WARNING: Kubernetes deployment or service file not found for module: ${module.trim()}"
                        }
                    }
                }
            }
        }

        /*stage('Deploy to Kubernetes') {
            steps {
                script {
                    def modules = env.CHANGED_MODULES.split(',')

                    modules.each { module ->
                        sh """
                            echo "Deploying module: ${module.trim()} to Kubernetes"
                            kubectl apply -f k8s/${module.trim()}/deployment.yaml
                            kubectl apply -f k8s/${module.trim()}/service.yaml
                        """
                    }
                }
            }
        }*/
    }
}
