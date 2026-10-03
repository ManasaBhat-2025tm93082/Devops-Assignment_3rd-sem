pipeline {
    agent any
    options { timestamps() }
    stages {
        stage('Checkout') {
            steps { checkout scm }
        }
        stage('Clean Build Environment') {
            steps {
                bat '''
                    if exist venv rmdir /s /q venv
                    python -m venv venv
                    venv\\Scripts\\python -m pip install --upgrade pip
                    venv\\Scripts\\python -m pip install -r requirements-dev.txt
                '''
            }
        }
        stage('Compile Check') {
            steps { bat 'venv\\Scripts\\python -m py_compile app.py' }
        }
        stage('Lint') {
            steps { bat 'venv\\Scripts\\python -m flake8 .' }
        }
        stage('Unit Tests') {
            steps { bat 'venv\\Scripts\\python -m pytest -v' }
        }
    }
    post {
        success { echo 'BUILD SUCCESSFUL' }
        failure { echo 'BUILD FAILED' }
    }
}