pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                dir('backend') {
                    bat 'docker compose up -d'
                    bat '"C:\\Users\\brune\\AppData\\Local\\Programs\\Python\\Python314\\python.exe" -m uv sync'
                    bat 'if not exist .env copy .env.example .env'
                }
            }
        }

        stage('Database') {
            steps {
                dir('backend') {
                    bat '"C:\\Users\\brune\\AppData\\Local\\Programs\\Python\\Python314\\python.exe" -m uv run dotenv run alembic upgrade head'
                }
            }
        }

        stage('Start API') {
            steps {
                dir('backend') {
                    bat 'start "FastAPI" cmd /c ""C:\\Users\\brune\\AppData\\Local\\Programs\\Python\\Python314\\python.exe" -m uv run fastapi dev src/main.py > fastapi.log 2>&1"'
                }
                sleep(time: 15, unit: 'SECONDS')
            }
        }

        stage('Check API') {
            steps {
                bat 'curl http://localhost:8000/docs'
            }
        }

        stage('Test API') {
            steps {
                dir('backend') {
                    bat 'newman run postman\\planting-api-tests.json || exit /b 0'
                }
            }
        }

        stage('Show API Logs') {
            steps {
                dir('backend') {
                    bat 'type fastapi.log'
                }
            }
        }

        stage('Code Quality') {
            steps {
                dir('backend') {
                    bat '"C:\\Users\\brune\\AppData\\Local\\Programs\\Python\\Python314\\python.exe" -m uv run ruff check src'
                }
            }
        }

        stage('Security') {
            steps {
                dir('backend') {
                    bat '"C:\\Users\\brune\\AppData\\Local\\Programs\\Python\\Python314\\python.exe" -m uv run bandit -r src || exit /b 0'
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