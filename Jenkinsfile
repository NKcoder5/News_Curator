// ─────────────────────────────────────────────────────────────────────────────
// News Curator – Jenkins Pipeline
// Triggered by GitHub Webhook on every push to main / master.
// Builds backend + frontend Docker images, pushes to Docker Registry,
// then deploys via docker-compose on the target server.
// ─────────────────────────────────────────────────────────────────────────────

pipeline {
    agent any

    // ── Environment ──────────────────────────────────────────────────────────
    environment {
        // Docker Hub or private registry
        DOCKER_REGISTRY   = "docker.io"                   // change to your registry
        DOCKER_NAMESPACE  = "your-dockerhub-username"     // change to your namespace
        BACKEND_IMAGE     = "${DOCKER_NAMESPACE}/news-curator-backend"
        FRONTEND_IMAGE    = "${DOCKER_NAMESPACE}/news-curator-frontend"

        // Jenkins Credential IDs (configure these in Jenkins → Credentials)
        DOCKER_CREDS      = credentials("dockerhub-credentials")   // username/password
        NEWS_API_KEY      = credentials("news-api-key")            // secret text
        JWT_SECRET        = credentials("jwt-secret")              // secret text
        REACT_APP_OPENWEATHER_API_KEY = credentials("openweather-api-key")

        // Tag images with the Git commit SHA for immutability + Git tag if present
        IMAGE_TAG         = "${env.GIT_COMMIT?.take(8) ?: 'latest'}"
    }

    // ── Triggers ─────────────────────────────────────────────────────────────
    triggers {
        // Listens for GitHub Webhook POST at <jenkins-url>/github-webhook/
        githubPush()
    }

    // ── Options ──────────────────────────────────────────────────────────────
    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: "10"))
        timeout(time: 30, unit: "MINUTES")
        disableConcurrentBuilds()
    }

    // ── Stages ────────────────────────────────────────────────────────────────
    stages {

        // 1. Checkout ──────────────────────────────────────────────────────────
        stage("Checkout") {
            steps {
                checkout scm
                echo "✅ Checked out branch: ${env.BRANCH_NAME} | commit: ${env.GIT_COMMIT}"
            }
        }

        // 2. Docker Login ──────────────────────────────────────────────────────
        stage("Docker Login") {
            steps {
                sh """
                    echo "${DOCKER_CREDS_PSW}" | \
                    docker login ${DOCKER_REGISTRY} -u "${DOCKER_CREDS_USR}" --password-stdin
                """
            }
        }

        // 3. Build Images (parallel) ───────────────────────────────────────────
        stage("Build Images") {
            parallel {

                stage("Build Backend") {
                    steps {
                        sh """
                            docker build \
                                -t ${BACKEND_IMAGE}:${IMAGE_TAG} \
                                -t ${BACKEND_IMAGE}:latest \
                                ./backend
                        """
                    }
                }

                stage("Build Frontend") {
                    steps {
                        sh """
                            docker build \
                                --build-arg REACT_APP_API_URL=http://backend:5000 \
                                --build-arg REACT_APP_OPENWEATHER_API_KEY=${REACT_APP_OPENWEATHER_API_KEY} \
                                -t ${FRONTEND_IMAGE}:${IMAGE_TAG} \
                                -t ${FRONTEND_IMAGE}:latest \
                                ./frontend
                        """
                    }
                }
            }
        }

        // 4. Push Images (parallel) ────────────────────────────────────────────
        stage("Push to Registry") {
            parallel {

                stage("Push Backend") {
                    steps {
                        sh """
                            docker push ${BACKEND_IMAGE}:${IMAGE_TAG}
                            docker push ${BACKEND_IMAGE}:latest
                        """
                    }
                }

                stage("Push Frontend") {
                    steps {
                        sh """
                            docker push ${FRONTEND_IMAGE}:${IMAGE_TAG}
                            docker push ${FRONTEND_IMAGE}:latest
                        """
                    }
                }
            }
        }

        // 5. Deploy via docker-compose ─────────────────────────────────────────
        stage("Deploy") {
            // Only deploy from the main/master branch
            when {
                anyOf {
                    branch "main"
                    branch "master"
                }
            }
            steps {
                // Write runtime .env so docker-compose can read secrets
                writeFile file: ".env.deploy", text: """
JWT_SECRET=${JWT_SECRET}
NEWS_API_KEY=${NEWS_API_KEY}
REACT_APP_OPENWEATHER_API_KEY=${REACT_APP_OPENWEATHER_API_KEY}
BACKEND_IMAGE=${BACKEND_IMAGE}:${IMAGE_TAG}
FRONTEND_IMAGE=${FRONTEND_IMAGE}:${IMAGE_TAG}
""".stripIndent()

                sh """
                    # Pull the freshly pushed images before recreating the containers
                    docker compose --env-file .env.deploy pull

                    # Bring services up (recreate changed containers, keep DB volume)
                    docker compose --env-file .env.deploy up -d --remove-orphans

                    # Clean up the un-tagged (dangling) images left behind
                    docker image prune -f

                    # Remove the temporary secrets file
                    rm -f .env.deploy
                """
            }
        }
    }

    // ── Post Actions ──────────────────────────────────────────────────────────
    post {
        always {
            sh "docker logout ${DOCKER_REGISTRY} || true"
            cleanWs()
        }
        success {
            echo "✅ Pipeline succeeded! Images: ${BACKEND_IMAGE}:${IMAGE_TAG} | ${FRONTEND_IMAGE}:${IMAGE_TAG}"
        }
        failure {
            echo "❌ Pipeline failed. Check the stage logs above for details."
        }
    }
}
