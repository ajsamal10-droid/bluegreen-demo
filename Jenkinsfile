pipeline {
    agent any

    parameters {
        string(name: 'APP_VERSION', defaultValue: '1', description: 'Version to deploy')
        booleanParam(name: 'BREAK_RELEASE', defaultValue: false, description: 'New version fails its smoke test')
        booleanParam(name: 'FAIL_AFTER_SWITCH', defaultValue: false, description: 'Force a failure after traffic is switched')
    }

    environment {
        BASE = '/opt/bluegreen'
        NET = 'bg-net'
        HEALTH_CODE = "${params.BREAK_RELEASE ? '500' : '200'}"
    }

    stages {
        stage('Prepare') {
            steps {
                script {
                    env.ACTIVE = sh(script: 'cat $BASE/active_color 2>/dev/null || echo none', returnStdout: true).trim()
                    env.NEW = (env.ACTIVE == 'blue') ? 'green' : 'blue'
                    env.SWITCHED = 'false'
                }
                sh 'docker network inspect $NET >/dev/null 2>&1 || docker network create $NET'
                echo "Live: ${env.ACTIVE} -> deploying to: ${env.NEW}"
            }
        }

        stage('Build') {
            steps {
                sh 'docker build -t bg-app:$APP_VERSION .'
            }
        }

        stage('Deploy to idle environment') {
            steps {
                sh '''
                    docker rm -f bg-$NEW 2>/dev/null || true
                    docker run -d --name bg-$NEW --network $NET -e COLOR=$NEW -e VERSION=$APP_VERSION -e HEALTH_CODE=$HEALTH_CODE bg-app:$APP_VERSION
                '''
            }
        }
        
        stage('Smoke test') {
            steps {
                sh '''
                    for i in 1 2 3 4 5; do
                      if docker exec bg-$NEW wget -q -O- http://127.0.0.1/health >/dev/null 2>&1 && docker exec bg-$NEW wget -q -O- http://127.0.0.1/ | grep -q "Version $APP_VERSION"; then
                        echo "Smoke test passed on bg-$NEW"
                        exit 0
                      fi
                      echo "Attempt $i failed, retrying..."
                      sleep 2
                    done
                    echo "Smoke test FAILED on bg-$NEW"
                    exit 1
                '''
            }
        }

        stage('Switch traffic') {
            steps {
                script { env.SWITCHED = 'true' }
                sh '''
                    mkdir -p $BASE/conf
                    echo "server { listen 80; location / { proxy_pass http://bg-$NEW:80; } }" > $BASE/conf/active.conf
                    if docker ps -q -f name=bg-router | grep -q .; then
                      docker exec bg-router nginx -t
                      docker exec bg-router nginx -s reload
                    else
                      docker rm -f bg-router 2>/dev/null || true
                      docker run -d --name bg-router --network $NET -p 8081:80 -v $BASE/conf:/etc/nginx/conf.d:ro nginx:alpine
                    fi
                    echo $NEW > $BASE/active_color
                '''
            }
        }

        stage('Verify live traffic') {
            steps {
                sh '''
                    sleep 2
                    docker exec bg-router wget -q -O- http://127.0.0.1/health
                    docker exec bg-router wget -q -O- http://127.0.0.1/ | grep "Version $APP_VERSION"
                '''
                script {
                    if (params.FAIL_AFTER_SWITCH) {
                        error('Simulated failure after switching traffic')
                    }
                }
            }
        }
            }

    post {
        success {
            slackSend(channel: '#jenkins-alerts', color: 'good',
                message: "SUCCESS: v${params.APP_VERSION} is live on ${env.NEW} (${env.JOB_NAME} #${env.BUILD_NUMBER})")
        }
        failure {
            script {
                stage('Rollback') {
                    sh '''
                        if [ "$SWITCHED" = "true" ] && [ "$ACTIVE" != "none" ]; then
                          echo "Rolling traffic back to $ACTIVE"
                          echo "server { listen 80; location / { proxy_pass http://bg-$ACTIVE:80; } }" > $BASE/conf/active.conf
                          docker exec bg-router nginx -s reload
                          echo $ACTIVE > $BASE/active_color
                          docker rm -f bg-$NEW 2>/dev/null || true
                        elif [ "$SWITCHED" = "true" ]; then
                          echo "First deployment failed after the switch, so there is no earlier version to roll back to"
                        else
                          echo "Traffic was never switched, removing the failed bg-$NEW"
                          docker rm -f bg-$NEW 2>/dev/null || true
                        fi
                    '''
                }
            }
            slackSend(channel: '#jenkins-alerts', color: 'danger',
                message: "FAILED and rolled back: ${env.JOB_NAME} #${env.BUILD_NUMBER}, live version unchanged\n${env.BUILD_URL}")
        }
    }
}
