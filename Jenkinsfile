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