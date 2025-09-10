pipeline {
    agent any

    environment {
        MANIFEST_REPO = "github.com/EAwangya/argocd-kubernetes.git"
        MANIFEST_DIR = "manifests"
        MANIFEST_FILE = "${MANIFEST_DIR}/deployment.yaml"
        TAG = "${env.BUILD_NUMBER}"
        DB_IMAGE = 'eawangya/myappdb'
        APP_IMAGE = 'eawangya/myapp'
        WEB_IMAGE = 'eawangya/myappweb'
        REPO = "EAwangya/argocd-kubernetes"
        BRANCH = "hotfix"
        BASE = "main"
    }

    options {
        timestamps()
    }

    stages {
        stage('Check Existing Pull Request') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'github-token', variable: 'TOKEN')]) {
                        def prExists = sh(
                            script: """
                                PR_LIST=\$(curl -s \
                                    -H "Accept: application/vnd.github+json" \
                                    -H "Authorization: Bearer \$TOKEN" \
                                    -H "X-GitHub-Api-Version: 2022-11-28" \
                                    "https://api.github.com/repos/${REPO}/pulls?state=open&head=${BRANCH}&base=${BASE}")
                                echo \$(echo "\$PR_LIST" | jq 'length')
                            """,
                            returnStdout: true
                        ).trim()

                        if (prExists != "0") {
                            error "Pull request already exists for branch '${BRANCH}' → exiting pipeline."
                        } else {
                            echo "No existing PR found — continuing pipeline..."
                        }
                    }
                }
            }
        }

        stage('Build & Push Docker Images') {
            steps {
                withDockerRegistry(credentialsId: 'dockerhub-creds', url: '') {
                    sh '''
                        # Ensure buildx exists
                        docker buildx create --name multiarch_builder --use || true
                        docker run --rm --privileged tonistiigi/binfmt --install all || true

                        # Build and push multi-arch images
                        docker buildx build --platform linux/amd64,linux/arm64 -t ${DB_IMAGE}:${TAG} -f database/Dockerfile ./database --push --progress=plain
                        docker buildx build --platform linux/amd64,linux/arm64 -t ${APP_IMAGE}:${TAG} -f app/Dockerfile ./app --push --progress=plain
                        docker buildx build --platform linux/amd64,linux/arm64 -t ${WEB_IMAGE}:${TAG} -f web/Dockerfile ./web --push --progress=plain
                    '''
                }
            }
        }

        stage('Clone or Pull GitHub Manifest Repo') {
            steps {
                script {
                    retry(3) {
                        if (!fileExists(MANIFEST_DIR)) {
                            sh "git clone -b ${BRANCH} https://${MANIFEST_REPO} ${MANIFEST_DIR}"
                        } else {
                            dir(MANIFEST_DIR) {
                                sh "git pull"
                                // sh "git fetch --all && git reset --hard origin/${BRANCH}"
                            }
                        }
                    }
                }
            }
        }

        stage('Update Kubernetes Manifest File') {
            steps {
                script {
                    dir(MANIFEST_DIR) {
                        // Use yq to safely update the image
                        // sh """
                        //     yq eval -i '.spec.template.spec.containers[] |= (select(.name=="app") .image = "${APP_IMAGE}:${TAG}")' ${MANIFEST_FILE}
                        // """
                        sh 'sed -i "s#eawangya/myapp:.*#${APP_IMAGE}:${TAG}#g" ${MANIFEST_FILE}'
                        sh 'sed -i "s#eawangya/myappdb:.*#${DB_IMAGE}:${TAG}#g" ${MANIFEST_FILE}'
                    }
                }
            }
        }

        stage('Commit and Push Changes') {
            steps {
                script {
                    dir(MANIFEST_DIR) {
                        withCredentials([usernamePassword(credentialsId: 'github-cred',
                                                         usernameVariable: 'GIT_USER',
                                                         passwordVariable: 'GIT_PASS')]) {
                            sh '''
                                git config user.email "ci-bot@example.com"
                                git config user.name "CI Bot"
                                git checkout -B ${BRANCH}
                                git add "${MANIFEST_FILE}"
                                git commit -m "Update App image to ${TAG}" || echo "No changes to commit"
                                git remote set-url origin https://${GIT_USER}:${GIT_PASS}@github.com/EAwangya/argocd-kubernetes.git
                                git push -u origin ${BRANCH} --force-with-lease || true
                            '''
                        }
                    }
                }
            }
        }

        stage('Create Pull Request') {
            steps {
                withCredentials([string(credentialsId: 'github-token', variable: 'TOKEN')]) {
                    sh """
                        curl -L -X POST \
                        -H "Accept: application/vnd.github+json" \
                        -H "Authorization: Bearer ${TOKEN}" \
                        -H "X-GitHub-Api-Version: 2022-11-28" \
                        https://api.github.com/repos/${REPO}/pulls \
                        -d '{"title":"Update App to v${TAG}","body":"Automated update from Jenkins","head":"${BRANCH}","base":"${BASE}"}' || true
                    """
                }
            }
        }
        }

    post {
        always {
            sh '''
                docker image prune -f || true
                docker builder prune -af || true
            '''
            archiveArtifacts artifacts: 'manifests/**,k8s/**', allowEmptyArchive: true, fingerprint: true
            cleanWs(deleteDirs: true, notFailBuild: true, patterns: [
                [pattern: 'manifests/**', type: 'EXCLUDE'],
                [pattern: 'k8s/**', type: 'EXCLUDE']
            ])
        }
        success { echo "Pipeline completed successfully!" }
        failure { echo "Pipeline failed. Check logs for details." }
    }
}
