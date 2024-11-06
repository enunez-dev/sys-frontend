pipeline {
    agent any

    parameters {
        separator(name:"GITHUB_CONFIGURATION", sectionHeader:"GITHUB CONFIGURATION")
        string(name: 'GITHUB_URL', defaultValue: 'https://github.com/enunez-dev/sys-frontend.git', description: 'GitHub URL project')
        string(name: 'GITHUB_BRANCH', defaultValue: 'master', description: 'Branch to deploy from')
        
        separator(name:"NGINX_CONFIGURATION", sectionHeader:"NGINX CONFIGURATION")        
        string(name: 'NGINX_EXECUTABLE_PATH', defaultValue: 'C:\\nginx\\nginx.exe', description: 'Set the path to Nginx executable')
        string(name: 'NGINX_BASE_PATH', defaultValue: 'C:\\nginx\\', description: 'Set the path to Nginx base')
        string(name: 'HTML_PATH', defaultValue: 'C:\\nginx\\html\\frontend', description: 'Set destination for frontend files')

        separator(name:"PUBLISH_CONFIGURATION", sectionHeader:"PUBLISH CONFIGURATION")    
        booleanParam(name: 'RUN_INSTALL', defaultValue: false, description: 'Check to run install dependecies')
        booleanParam(name: 'RUN_TESTS', defaultValue: false, description: 'Check to run tests')
        string(name: 'OUTPUT_FOLDER', defaultValue: 'OutputBuilds', description: 'Folder contain output build app')
        choice(name: 'ENVIRONMENT', choices: ['Development','Testing','Staging','Production'], description: 'Target environment')

    }

    environment {
        BUILD_PATH = "${params.OUTPUT_FOLDER}" + "\\" + "${BUILD_NUMBER}" + "\\" + "${params.ENVIRONMENT}"
    }

    stages {
        stage('Checkout') {
            steps {
                git(branch: "${params.GITHUB_BRANCH}", url: "${params.GITHUB_URL}")
            }
        }

        stage('Install Dependencies') {
            when {
                expression { params.RUN_INSTALL }
            }
            steps {
                bat 'npm install'
            }
        }
        stage('Run Test') {
            when {
                expression { params.RUN_TESTS }
            }
            steps {
                bat 'npm run test'
            }
        }

        stage('Down Service Nginx') {
            steps {
                script {
                    // Check if Nginx is running, then stop it if it is
                    def isRunning = bat(script: 'tasklist | findstr /I nginx.exe', returnStatus: true) == 0
                    if (isRunning) {
                        echo 'Nginx is running; stopping the service for app update.'
                        bat "\"${params.NGINX_EXECUTABLE_PATH}\" -p ${params.NGINX_BASE_PATH} -s stop"
                    } else {
                        echo 'Nginx is not running; proceeding with app update.'
                    }
                }
            }
        }

        stage('Build with Vite') {
            steps {
                script {
                    bat "npm run build -- --outDir ${env.BUILD_PATH}"
                }
            }
        }

        stage('Publish build to server') {
            steps {
                script {
                    def workspacePath = "${env.WORKSPACE}\\${env.BUILD_PATH}"
                    bat "xcopy /E /I /Y \"${workspacePath}\" \"${params.HTML_PATH}\""
                }
            }
        }
        stage('Push artifacts') {
            steps {
                script {
                    def timestamp = new java.text.SimpleDateFormat("yyyyMMddhhmmss")
                    String today = timestamp.format(new Date())
                    String filesPath = "${params.OUTPUT_FOLDER}" + "\\" + "${BUILD_NUMBER}"
                    String outputPath = "${env.BUILD_PATH}" + "\\"+today+"_" + "${BUILD_NUMBER}" + "_outputFiles.zip"
                    
                    zip zipFile: outputPath, archive:false, dir:filesPath
                    archiveArtifacts artifacts:outputPath, caseSensitive: false
                }
            }
        }

        stage('Up server nginx') {
            steps {
                script {
                    withEnv(['JENKINS_NODE_COOKIE=do_not_kill']) {
                        bat "start /B cmd /c \"${params.NGINX_EXECUTABLE_PATH}\" -p ${params.NGINX_BASE_PATH}"
                        echo 'Nginx is now running after app update.'
                        bat(script: 'tasklist | findstr /I nginx.exe', returnStatus: true)
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Frontend deployment process completed.'            
        }
        failure {
            echo 'Deployment failed, please check the logs for more details.'
        }
    }
}