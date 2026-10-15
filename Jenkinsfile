pipeline {
    agent any

    options {
        skipDefaultCheckout()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '15'))
    }

    triggers {
        pollSCM('H/2 * * * *')
    }

    environment {
        APP_NAME   = 'hospital-portal'
        IMAGE_REPO = 'adityarajarora123/hospital-portal'
        IMAGE_TAG  = "v1.0.${env.BUILD_NUMBER}"
        NODE_PORT  = '30094'
        DOCKERHUB  = credentials('dockerhub')
        KUBECONFIG = credentials('kubeconfig')
    }

    stages {
        stage('Clone Code') {
            steps {
                checkout scm
                script {
                    run 'git log -1 --oneline'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    run "docker build --build-arg APP_VERSION=${IMAGE_TAG} -t ${IMAGE_REPO}:${IMAGE_TAG} ."
                }
            }
        }

        stage('Test Application') {
            steps {
                script {
                    run "docker image inspect ${IMAGE_REPO}:${IMAGE_TAG} --format \"Image built: {{.Id}} ({{.Size}} bytes)\""
                    run "docker run --rm ${IMAGE_REPO}:${IMAGE_TAG} nginx -t"
                    run "docker run --rm ${IMAGE_REPO}:${IMAGE_TAG} test -s /usr/share/nginx/html/index.html"
                    run "docker run --rm ${IMAGE_REPO}:${IMAGE_TAG} cat /usr/share/nginx/html/version.txt"
                }
            }
        }

        stage('Push Image') {
            steps {
                script {
                    dockerLogin()
                    run "docker push ${IMAGE_REPO}:${IMAGE_TAG}"
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    renderManifest('deployment.yaml', 'deployment.rendered.yaml')
                    run 'kubectl apply -f service.yaml'
                    run 'kubectl apply -f deployment.rendered.yaml'
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                script {
                    boolean healthy = runStatus("kubectl rollout status deployment/${APP_NAME} --timeout=120s") == 0
                    if (healthy) {
                        healthy = runStatus("kubectl exec -i deployment/${APP_NAME} -- sh -s http://${APP_NAME} < ${localPath('tests/smoke-test.sh')}") == 0
                    }
                    if (!healthy) {
                        rollbackRelease(APP_NAME)
                        error "Release ${IMAGE_TAG} failed verification and was rolled back to the previous stable revision."
                    }
                    run "kubectl get deployment,pods,service -l app=${APP_NAME} -o wide"
                }
            }
        }

        stage('Promote Stable Release') {
            steps {
                script {
                    run "docker tag ${IMAGE_REPO}:${IMAGE_TAG} ${IMAGE_REPO}:latest"
                    run "docker push ${IMAGE_REPO}:latest"
                    run "kubectl rollout history deployment/${APP_NAME}"
                }
            }
        }
    }

    post {
        success {
            echo "Release ${IMAGE_TAG} is live at http://localhost:${NODE_PORT}"
        }
        failure {
            echo "Pipeline failed for release ${IMAGE_TAG}. Check the stage logs above."
        }
        always {
            script { runStatus('docker logout') }
        }
    }
}

def run(String command) {
    if (isUnix()) {
        sh command
    } else {
        bat command
    }
}

def runStatus(String command) {
    return isUnix() ? sh(script: command, returnStatus: true) : bat(script: command, returnStatus: true)
}

def localPath(String path) {
    return isUnix() ? path : path.replace('/', '\\')
}

def dockerLogin() {
    if (isUnix()) {
        sh 'echo "$DOCKERHUB_PSW" | docker login -u "$DOCKERHUB_USR" --password-stdin'
    } else {
        bat '@echo %DOCKERHUB_PSW%| docker login -u %DOCKERHUB_USR% --password-stdin'
    }
}

def renderManifest(String source, String target) {
    String manifest = readFile(source)
        .replace("${env.IMAGE_REPO}:latest", "${env.IMAGE_REPO}:${env.IMAGE_TAG}")
        .replace('Initial release', "Build #${env.BUILD_NUMBER} - ${env.IMAGE_TAG}")
    writeFile file: target, text: manifest
}

def rollbackRelease(String deployment) {
    echo "Rollout of ${env.IMAGE_TAG} failed. Collecting diagnostics before rolling back."
    runStatus("kubectl get pods -l app=${deployment} -o wide")
    runStatus('kubectl get events --field-selector type=Warning --sort-by=.lastTimestamp')
    run "kubectl rollout undo deployment/${deployment}"
    run "kubectl rollout status deployment/${deployment} --timeout=120s"
    run "kubectl rollout history deployment/${deployment}"
    run "kubectl get pods -l app=${deployment} -o wide"
}
