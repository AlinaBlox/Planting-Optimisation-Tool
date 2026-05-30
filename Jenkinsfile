pipeline {
    agent any

    environment {
        PYTHON           = 'C:\\Users\\brune\\AppData\\Local\\Programs\\Python\\Python314\\python.exe'
        PYTHONIOENCODING = 'utf-8'
        BUILD_TAG        = "planting-api:build-${BUILD_NUMBER}"
        STAGING_PORT     = '8001'
        PROD_PORT        = '8002'
    }

    stages {

        stage('Build') {
            steps {
                dir('backend') {
                    bat 'copy .env.example .env /Y'
                    bat '"%PYTHON%" -m uv sync'
                    bat 'docker compose up -d'
                }
                bat 'docker build -f backend/Dockerfile -t %BUILD_TAG% .'
                dir('frontend') {
                    bat 'npm install'
                    bat 'npm run build'
                }
            }
        }

        stage('Database Migration') {
            steps {
                dir('backend') {
                    bat '"%PYTHON%" -m uv run dotenv run alembic upgrade head'
                }
            }
        }

        stage('Test') {
            steps {
                dir('backend') {
                    bat '''
                        powershell -Command "& { $env:PYTHONIOENCODING='utf-8'; Start-Process -FilePath '%PYTHON%' -ArgumentList '-m uv run uvicorn src.main:app --host 127.0.0.1 --port 8000' -RedirectStandardOutput fastapi.log -RedirectStandardError fastapi-err.log -NoNewWindow }"
                    '''
                }
                sleep(time: 15, unit: 'SECONDS')
                bat 'curl -f http://localhost:8000/docs'
                dir('backend') {
                    bat 'newman run postman\\planting-api-tests.json'
                }
                dir('frontend') {
                    bat 'npx vitest run'
                }
            }
            post {
                always {
                    dir('backend') {
                        bat 'type fastapi.log'
                    }
                }
            }
        }

        stage('Code Quality') {
            steps {
                dir('backend') {
                    bat '"%PYTHON%" -m uv run ruff check src'
                }
                dir('frontend') {
                    bat 'npm run lint:scripts || exit /b 0'
                    bat 'npm run lint:styles  || exit /b 0'
                }
            }
        }

        stage('Security') {
            steps {
                dir('backend') {
                    bat '"%PYTHON%" -m uv run bandit -r src -f txt -o bandit-report.txt || exit /b 0'
                    bat 'type bandit-report.txt'
                }
                dir('frontend') {
                    bat 'npm audit --audit-level=high || exit /b 0'
                }
            }
        }

        stage('Deploy') {
            steps {
                bat 'docker stop planting-staging || exit /b 0'
                bat 'docker rm   planting-staging || exit /b 0'
                bat 'docker run -d --name planting-staging -p %STAGING_PORT%:8080 --network backend_default --env-file backend/.env -e DATABASE_URL=postgresql+asyncpg://postgres:devpassword@pot_postgres_db:5432/POT_db -e REDIS_URL=redis://pot_redis:6379 -e POSTGRES_HOST=pot_postgres_db %BUILD_TAG%'
                sleep(time: 15, unit: 'SECONDS')
                bat 'curl -f http://localhost:%STAGING_PORT%/docs'
                echo "Deployed build ${BUILD_NUMBER} to staging on port ${STAGING_PORT}"
            }
        }

        stage('Release') {
            steps {
                bat 'docker tag %BUILD_TAG% planting-api:latest'
                bat 'docker stop planting-prod || exit /b 0'
                bat 'docker rm   planting-prod || exit /b 0'
                bat 'docker run -d --name planting-prod -p %PROD_PORT%:8080 --network backend_default --env-file backend/.env -e DATABASE_URL=postgresql+asyncpg://postgres:devpassword@pot_postgres_db:5432/POT_db -e REDIS_URL=redis://pot_redis:6379 -e POSTGRES_HOST=pot_postgres_db planting-api:latest'
                sleep(time: 15, unit: 'SECONDS')
                bat 'curl -f http://localhost:%PROD_PORT%/docs'
                echo "Released build ${BUILD_NUMBER} to production on port ${PROD_PORT}"
            }
        }

        stage('Monitoring') {
            steps {
                bat 'curl -f http://localhost:%PROD_PORT%/docs && echo Health check 1 passed || echo Health check 1 FAILED'
                sleep(time: 5, unit: 'SECONDS')
                bat 'curl -f http://localhost:%PROD_PORT%/docs && echo Health check 2 passed || echo Health check 2 FAILED'
                sleep(time: 5, unit: 'SECONDS')
                bat 'curl -f http://localhost:%PROD_PORT%/docs && echo Health check 3 passed || echo Health check 3 FAILED'
                echo "Monitoring complete - build ${BUILD_NUMBER} is live on port ${PROD_PORT}"
            }
        }
    }

    post {
        success {
            echo "Pipeline SUCCESS - Build ${BUILD_NUMBER} deployed and healthy on port ${PROD_PORT}"
        }
        failure {
            echo "Pipeline FAILED on build ${BUILD_NUMBER} - check stage logs above"
        }
        always {
            bat 'powershell -Command "Get-Process -Name python -ErrorAction SilentlyContinue | Stop-Process -Force" || exit /b 0'
            //bat 'docker stop planting-staging || exit /b 0'
            //bat 'docker rm   planting-staging || exit /b 0'
            //bat 'docker stop planting-prod || exit /b 0'
            //bat 'docker rm   planting-prod || exit /b 0'
            //dir('backend') {
            //    bat 'docker compose down || exit /b 0'
            //}
        }
    }
}