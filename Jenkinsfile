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
