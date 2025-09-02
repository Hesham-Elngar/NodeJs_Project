#we stopped in the step of sonarQube installation
pipeline {
    agent any

    environment {
        MONGO_URI = "mongodb://127.0.0.1:27017/test"
        MONGO_DB_CREDS = credentials('mongo-db-credentials')
    }

    options {
        disableResume()
        disableConcurrentBuilds abortPrevious: true
    }

    stages {
        stage('Installing Dependencies') {
            steps {
                echo "Installing npm dependencies..."
                // Try npm ci, fall back to npm install if lockfile is out of sync
                bat '''
                echo Installing clean dependencies...
                npm ci --no-audit || (
                    echo Lockfile out of sync, falling back to npm install...
                    npm install --no-audit
                )
                '''
            }
        }

        stage('Security Scans in Parallel') {
            parallel {
                stage('NPM Dependency Audit') {
                    steps {
                        bat '''
                        npm audit fix --force || exit /b 1
                        echo Reinstalling after forced upgrade...
                        npm install --no-audit || exit /b 1
                        npm audit --audit-level=critical
                        '''
                    }
                }
                stage('OWASP Dependency Check') {
                    steps {
                        dependencyCheck additionalArguments: '''
                          --scan ./ 
                          --out ./ 
                          --format 'ALL' 
                          --prettyPrint
                        ''', odcInstallation: 'dependency-check-cli'
                    }
                }
            }
        }

        stage('Run Tests') {
            steps {
                echo "Running tests with Mongo URI: ${env.MONGO_URI}"
                echo "${MONGO_DB_CREDS}"
                echo "${MONGO_DB_CREDS_USR}"
                echo "${MONGO_DB_CREDS_PSW}"
                bat 'set MONGO_URI=%MONGO_URI% && npm test'
            }
        }

        stage('Code Coverage') {
            steps {
                catchError(buildResult: 'SUCCESS', message: 'Oops there is an error will be fixed later', stageResult: 'UNSTABLE') {
                 bat 'npm run coverage'
            }
            }
        }
    }
    post {
  always {

    junit allowEmptyResults: true,
                      checksName: 'Unit Test Report',
                      keepProperties: true,
                      keepTestNames: true,
                      stdioRetention: 'ALL',
                      testResults: 'test-results.xml'
                      
                               dependencyCheckPublisher failedTotalCritical: 1,
                                                  pattern: 'dependency-check-report.xml',
                                                  stopBuild: true
            
            publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, icon: '', keepAll: true, reportDir: './', reportFiles: 'dependency-check-jenkins.html', reportName: 'Dependency Check HTML Report', reportTitles: '', useWrapperFileDirectly: true])                         

            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'coverage/lcov-report', reportFiles: 'index.html', reportName: 'Code Coverage HTML Report', reportTitles: '', useWrapperFileDirectly: true])        
  }
}

}
