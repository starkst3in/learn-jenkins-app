pipeline {
    agent any
    stages {
        stage('Build'){
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps{
                sh '''
                    ls -la
                    node --version
                    npm --version
                    npm ci --cache .npm --prefer-offline || { echo "=== npm ci failed, dumping debug log ==="; cat /home/node/.npm/_logs/*-debug-0.log 2>/dev/null; cat .npm/_logs/*-debug-0.log 2>/dev/null; exit 1; }
                    npm run build
                    ls -la
                
                '''
            }
        }
    }
}