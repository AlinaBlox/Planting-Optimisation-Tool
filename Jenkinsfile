pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out repository from GitHub'
            }
        }

        stage('Build') {
            steps {
                dir('backend') {
                    bat 'docker compose up -d'
                    bat '"C:\\Users\\brune\\AppData\\Local\\Programs\\Python\\Python314\\python.exe" -m uv sync'
                    bat 'copy .env.example .env'
                }
            }
        }

        stage('Database Migration') {
            steps {
                dir('backend') {
                    bat '"C:\\Users\\brune\\AppData\\Local\\Programs\\Python\\Python314\\python.exe" -m uv run dotenv run alembic upgrade head'
                }
            }
        }

        stage('Start API') {
            steps {
                dir('backend') {
                    bat 'start /B "FastAPI" "C:\\Users\\brune\\AppData\\Local\\Programs\\Python\\Python314\\python.exe" -m uv run fastapi dev src/main.py'
                    bat 'timeout /t 10'
                }
             }
        }
        
        stage('API Test with Newman') {
            steps {
                dir('backend') {
                    bat 'newman run postman\\planting-api-tests.json'
                }
            }
        }
    }

    post {
        always {
            dir('backend') {
                bat 'docker compose down'
            }
        }
    }
}