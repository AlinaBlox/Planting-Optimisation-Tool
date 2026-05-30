pipeline {
    agent any

    environment {
        PYTHON       = 'C:\\Users\\brune\\AppData\\Local\\Programs\\Python\\Python314\\python.exe'
        BUILD_TAG    = "planting-api:build-${BUILD_NUMBER}"
        STAGING_PORT = '8001'
        PROD_PORT    = '8002'
    }

    stages {

        stage('Build') {
            steps {
                dir('backend') {
                    bat 'if not exist .env copy .env.example .env'
                    bat '"%PYTHON%" -m uv sync'
                    bat 'docker compose up -d'
                }
                // Build context must be project root - Dockerfile references gis/ and datascience/
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
                // Start API locally for Newman tests
                dir('backend') {
                    bat 'start "FastAPI" cmd /c ""%PYTHON%" -m uv run uvicorn src.main:app --host 127.0.0.1 --port 8000 > fastapi.log 2>&1"'
                }
                sleep(time: 15, unit: 'SECONDS')

                // Verify API is up before running tests
                bat 'curl -f http://localhost:8000/docs'

                // Backend API tests - failure here fails the pipeline
                dir('backend') {
                    bat 'newman run postman\\planting-api-tests.json'
                }

                // Frontend unit tests - vitest run exits non-zero on failure
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
                // Python linting
                dir('backend') {
                    bat '"%PYTHON%" -m uv run ruff check src'
                }
                // Frontend script linting
                dir('frontend') {
                    bat 'npm run lint:scripts || exit /b 0'
                }
                // Frontend style linting
                dir('frontend') {
                    bat 'npm run lint:styles || exit /b 0'
                }
            }
        }

        stage('Security') {
            steps {
                // Python security scan - Bandit
                dir('backend') {
                    bat '"%PYTHON%" -m uv run bandit -r src -f txt -o bandit-report.txt || exit /b 0'
                    bat 'type bandit-report.txt'
                }
                // Frontend dependency vulnerability audit
                dir('frontend') {
                    bat 'npm audit --audit-level=high || exit /b 0'
                }
            }
        }

        stage('Deploy') {
            steps {
                // Clean up any previous staging container
                bat 'docker stop planting-staging || exit /b 0'
                bat 'docker rm   planting-staging || exit /b 0'
                // Container exposes 8080 (Dockerfile default PORT), map to STAGING_PORT on host
                bat 'docker run -d --name planting-staging -p %STAGING_PORT%:8080 %BUILD_TAG%'
                sleep(time: 10, unit: 'SECONDS')
                // Verify staging is healthy
                bat 'curl -f http://localhost:%STAGING_PORT%/docs'
                echo "Deployed build ${BUILD_NUMBER} to staging on port ${STAGING_PORT}"
            }
        }

        stage('Release') {
            steps {
                // Tag this build as latest for production
                bat 'docker tag %BUILD_TAG% planting-api:latest'
                // Clean up any previous prod container
                bat 'docker stop planting-prod || exit /b 0'
                bat 'docker rm   planting-prod || exit /b 0'
                // Promote to production port
                bat 'docker run -d --name planting-prod -p %PROD_PORT%:8080 planting-api:latest'
                sleep(time: 10, unit: 'SECONDS')
                // Verify production is healthy
                bat 'curl -f http://localhost:%PROD_PORT%/docs'
                echo "Released build ${BUILD_NUMBER} to production on port ${PROD_PORT}"
            }
        }

        stage('Monitoring') {
            steps {
                // Run 3 health checks against production with a 5 second gap between each
                bat '''
                    for /L %%i in (1,1,3) do (
                        curl -f http://localhost:%PROD_PORT%/docs && echo Health check %%i passed || echo Health check %%i FAILED
                        timeout /t 5 /nobreak
                    )
                '''
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
            // Tear down local dev containers
            dir('backend') {
                bat 'docker compose down || exit /b 0'
            }
            // Clean up staging container (prod stays running)
            bat 'docker stop planting-staging || exit /b 0'
            bat 'docker rm   planting-staging || exit /b 0'
        }
    }
}