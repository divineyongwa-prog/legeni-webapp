// Lots 4 et 5 : intégration continue + déploiement continu de Legeni WebApp
pipeline {
    agent any

    environment {
        IMAGE = 'divine2026/legeni-webapp'
    }

    options {
        timestamps()
        timeout(time: 45, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '15'))
    }

    stages {
        stage('Version') {
            steps {
                script {
                    env.VERSION = readFile('VERSION').trim()
                    env.TAG = "${env.VERSION}-${env.BUILD_NUMBER}"
                    currentBuild.displayName = "#${env.BUILD_NUMBER} ${env.TAG}"
                }
            }
        }

        stage('Contrôles qualité') {
            parallel {
                stage('flake8 + tests + couverture') {
                    steps {
                        sh '''
                            mkdir -p reports
                            docker build --target test -t legeni-test:$BUILD_NUMBER .
                            docker rm -f pytest-$BUILD_NUMBER 2>/dev/null || true
                            set +e
                            docker run --name pytest-$BUILD_NUMBER legeni-test:$BUILD_NUMBER \
                                pytest -q --cov=app --cov-report=xml:/tmp/coverage.xml --cov-report=term \
                                --cov-fail-under=80 --junitxml=/tmp/junit.xml
                            RC=$?
                            docker cp pytest-$BUILD_NUMBER:/tmp/junit.xml reports/junit.xml
                            docker cp pytest-$BUILD_NUMBER:/tmp/coverage.xml reports/coverage.xml
                            docker rm -f pytest-$BUILD_NUMBER
                            exit $RC
                        '''
                    }
                    post {
                        always {
                            junit allowEmptyResults: true, testResults: 'reports/junit.xml'
                            catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                                recordCoverage tools: [[parser: 'COBERTURA', pattern: 'reports/coverage.xml']]
                            }
                        }
                    }
                }
                stage('hadolint') {
                    steps {
                        sh 'docker run --rm -i hadolint/hadolint < Dockerfile'
                    }
                }
                stage('Syntaxe Ansible') {
                    steps {
                        withCredentials([file(credentialsId: 'ansible-vault-pass', variable: 'VAULT_PASS')]) {
                            dir('ansible') {
                                sh '''
                                    ansible-playbook provision.yml --syntax-check --vault-password-file $VAULT_PASS
                                    ansible-playbook deploy.yml --syntax-check --vault-password-file $VAULT_PASS \
                                        -e target_env=staging -e app_version=0.0.0-0
                                '''
                            }
                        }
                    }
                }
            }
        }

        stage('Build image') {
            steps {
                sh 'docker build --build-arg APP_VERSION=$TAG -t $IMAGE:$TAG .'
            }
        }

        stage('Scan Trivy') {
            steps {
                sh '''
                    docker run --rm -v trivy-cache:/root/.cache/ -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy image --timeout 30m --exit-code 1 --severity CRITICAL --ignore-unfixed $IMAGE:$TAG
                '''
            }
        }

        stage('Push Docker Hub') {
            when { branch 'main' }
            steps {
                withCredentials([usernamePassword(credentialsId: 'registry-credentials',
                                                  usernameVariable: 'DH_USER',
                                                  passwordVariable: 'DH_TOKEN')]) {
                    sh '''
                        echo "$DH_TOKEN" | docker login -u "$DH_USER" --password-stdin
                        docker push $IMAGE:$TAG
                    '''
                }
            }
        }

        stage('Déploiement staging') {
            when { branch 'main' }
            steps {
                script { deployTo('staging') }
            }
        }

        stage('Validation production') {
            when { branch 'main' }
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input message: "Déployer ${env.TAG} en production ?", ok: 'Déployer'
                }
            }
        }

        stage('Déploiement production') {
            when { branch 'main' }
            steps {
                script { deployTo('prod') }
            }
        }
    }

    post {
        success {
            echo "Version ${env.TAG} : pipeline terminé avec succès"
        }
        failure {
            script { notify("Échec du pipeline ${env.JOB_NAME} #${env.BUILD_NUMBER} (${env.TAG}) : ${env.BUILD_URL}") }
        }
        always {
            sh 'docker image rm legeni-test:$BUILD_NUMBER 2>/dev/null || true'
        }
    }
}

// Déploiement par Ansible : rollback automatique et smoke test inclus dans deploy.yml
def deployTo(String envName) {
    withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG'),
                     file(credentialsId: 'ansible-vault-pass', variable: 'VAULT_PASS')]) {
        dir('ansible') {
            sh "ansible-playbook deploy.yml -e target_env=${envName} -e app_version=${env.TAG} -e deploy_timeout=300 --vault-password-file \$VAULT_PASS"
        }
    }
}

// Notification Discord ou Slack (facultative : sans credential 'notify-webhook', on affiche seulement un message)
def notify(String message) {
    try {
        withCredentials([string(credentialsId: 'notify-webhook', variable: 'HOOK')]) {
            writeJSON file: 'notify.json', json: [content: message, text: message]
            sh 'curl -s -H "Content-Type: application/json" -d @notify.json "$HOOK" || true'
        }
    } catch (err) {
        echo "Notification non envoyée : ${err.message}"
    }
}
