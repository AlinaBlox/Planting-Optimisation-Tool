pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                dir('backend') {
                    bat 'docker compose up -d'
                    bat '"C:\\Users\\brune\\AppData\\Local\\Programs\\Python\\Python314\\python.exe" -m uv sync'
                    bat 'copy .env.example .env'
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

        stage('Seed Test Data') {
            steps {
                dir('backend') {
                    bat 'set PYTHONPATH=.&& "C:\\Users\\brune\\AppData\\Local\\Programs\\Python\\Python314\\python.exe" -m uv run python src\\scripts\\create_test_user.py'
                    bat 'set PYTHONPATH=.&& "C:\\Users\\brune\\AppData\\Local\\Programs\\Python\\Python314\\python.exe" -m uv run python src\\scripts\\seed_references.py'
                }
            }
        }

        stage('Start API') {
            steps {
                 dir('backend') {
                    bat 'start /B "" "C:\\Users\\brune\\AppData\\Local\\Programs\\Python\\Python314\\python.exe" -m uv run fastapi dev src/main.py'
                    sleep(time: 10, unit: 'SECONDS')
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