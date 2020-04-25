def container
def commit
def commitChangeset
def url

pipeline {

    agent any

    options {
        preserveStashes(buildCount: 10) 
        buildDiscarder(logRotator(numToKeepStr:'10')) 
    }

    environment {
        JKS_USERID = sh(script:"""id -u jenkins """, returnStdout: true).trim()
        JKS_GROUPID = sh(script:"""id -g jenkins """, returnStdout: true).trim()
        APP_NAME = readJSON(file: 'composer.json').name.trim()
        APP_VERSION = readJSON(file: 'composer.json').version.trim()
        SONAR_PROJECT_KEY = "marcosborges_baseline_graphql_siler"
        SONAR_ORGANIZATION_KEY = "baseline-graphql-siler"
        REGISTRY_HOST = credentials('REGISTRY_HOST')
        GOOGLE_APPLICATION_CREDENTIALS = credentials('GCP_SERVICE_ACCOUNT')
        GOOGLE_REGION = "us-east1"
        GOOGLE_ZONE = "us-east1-a"
    }

    stages {

        stage('Checkout') {
            steps {
                //checkout scm
                script {
                    commit = sh(returnStdout: true, script: 'git rev-parse --short=8 HEAD').trim()
                    commitChangeset = sh(returnStdout: true, script: 'git diff-tree --no-commit-id --name-status -r HEAD').trim()
                }
                stash includes: '**/*', name: 'checkoutSources'
            }
            post {
                failure {
                    echo 'Falha ao executar o checkout do projeto :('
                }
            }
        }

        stage('Restore') {
            agent {
                docker { image 'phpswoole/swoole' }
            }
            steps {
                sh " composer -q -n install "
                stash includes: 'vendor/**/*', name: 'restoreSources'
            }
            post {
                failure {
                    echo 'Falha ao executar o restauração de dependencias :('
                }
            }
        }

        stage('Test') {
            agent {
                dockerfile { 
                    filename 'Dockerfile'
                    dir './'
                }
            }
            steps {
                unstash 'restoreSources'
                sh " composer test "
                sh """
                    sed -i 's|${pwd()}/||' ${pwd()}/tests/_reports/logs/clover.xml 
                    sed -i 's|${pwd()}/||' ${pwd()}/tests/_reports/logs/junit.xml
                """
                stash includes: '**/*', name: 'testSources'
            }
            post {
                failure {
                    echo 'Falha ao executar os testes :('
                }
            }
        }

        stage('Review') {
            steps {
                unstash 'checkoutSources'
                unstash 'restoreSources'
                unstash 'testSources'
              
                script {
                    def scannerHome = tool 'SonarScanner';
                    withSonarQubeEnv ('SonarQubeCloud') {
                        sh """    
                            ${scannerHome}/bin/sonar-scanner \
                                -Dsonar.projectKey="${env.SONAR_PROJECT_KEY}" \
                                -Dsonar.organization="${env.SONAR_ORGANIZATION_KEY}" \
                                -Dsonar.projectName="${env.APP_NAME}" \
                                -Dsonar.projectVersion="${env.APP_VERSION}" \
                                -Dsonar.sources="src" \
                                -Dsonar.tests="tests" \
                                -Dsonar.language="php" \
                                -Dsonar.sourceEncoding="UTF-8" \
                                -Dsonar.php.coverage.reportPaths=tests/_reports/logs/clover.xml \
                                -Dsonar.php.tests.reportPath=tests/_reports/logs/junit.xml \
                        """
                    }
                }
                timeout (time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
            post {
                failure {
                    echo 'Falha ao executar a revisão de código e cobertura de testes :('
                }
            }
        }

        stage('Build') {
            steps {
                script {
                    unstash 'restoreSources'
                    container = docker.build("${env.REGISTRY_HOST}${env.APP_NAME}:${env.APP_VERSION}", " -f Build.Dockerfile . ")
                }
            }
            post {
                failure {
                    echo 'Falha ao executar a construção do container :('
                }
            }
        }

        stage('Registry') {
            steps {
                script {
                    sh script:'#!/bin/sh -e\n' +  """ docker login -u _json_key -p "\$(cat ${env.GOOGLE_APPLICATION_CREDENTIALS})" https://${env.REGISTRY_HOST}""", returnStdout: false
                    docker.withRegistry("https://${env.REGISTRY_HOST}") {
                        container.push("${env.APP_VERSION}")
                        container.push("${commit}")
                        container.push("latest")
                    }
                }
            }
            post {
                failure {
                    echo 'Falha ao registrar o container :('
                }
            }
        }  

        stage('Approval') {
            steps {
                script {
                    timeout(time: 2, unit: 'HOURS') {
                        input message: 'Approve Deploy?', ok: 'Yes'
                    }
                }
            }
        }
                
        stage('Deploy') {
            when {
                expression {
                    currentBuild.result == null || currentBuild.result == 'SUCCESS' 
                }
            }
            steps {
                script {
                    def data = readJSON file: env.GOOGLE_APPLICATION_CREDENTIALS 
                    def _name = env.APP_NAME.toLowerCase().replace('_','-').replace('/','-').replace('.','-')
                    sh """
                        export GOOGLE_APPLICATION_CREDENTIALS=${env.GOOGLE_APPLICATION_CREDENTIALS}
                        gcloud config set project ${data.project_id}
                        gcloud config set compute/zone ${env.GOOGLE_ZONE}
                        gcloud auth activate-service-account ${data.client_email} --key-file=${env.GOOGLE_APPLICATION_CREDENTIALS} --project=${data.project_id}
                        gcloud run deploy ${_name} \
                            --image ${env.REGISTRY_HOST}${env.APP_NAME}:${env.APP_VERSION} \
                            --platform managed \
                            --memory 2Gi \
                            --concurrency 10 \
                            --timeout 1m20s \
                            --max-instances 2 \
                            --cpu 1000m \
                            --port 9501 \
                            --labels "name=${_name}" \
                            --region ${env.GOOGLE_REGION} \
                            --allow-unauthenticated \
                            --set-env-vars "APP_ENV=development"
                    """
                    
                    def service = readJSON(text: sh(script: """
                        gcloud run services describe ${_name} \
                            --platform managed \
                            --region ${env.GOOGLE_REGION} \
                            --format json
                        """, returnStdout : true).trim()
                    )

                    url = service.status.address.url

                }
            }
            post {
                failure {
                    echo 'Falha ao realizar o deploy :('
                }
            }
        }

        stage('Validate') {
            steps {
                script {
                    sh """ curl -X POST -H "Content-type: application/json" -d '{"query": "query{helloWorld}"}' ${url}/graphql """
                    echo "Aplicação publicada cm sucesso: ${url}" 
                }
            }
            post {
                failure {
                    echo 'Falha ao realizar o deploy :('
                }
            }
        }

    }

    post {
        success {
            echo 'The Pipeline success :)'
            
            script {
                sh "zip -r dist.zip ./"
            }

            junit "tests/_reports/**/*.xml"

            archiveArtifacts artifacts: 'dist.zip', fingerprint: true
            
            /*
            allure([
                includeProperties: false,
                jdk: 'JDK8',
                properties: [],
                reportBuildPolicy: 'ALWAYS',
                results: [
                    [path: "tests/_reports/logs/"]
                ]
            ])
            */
            
            publishHTML target: [
                allowMissing: false,
                alwaysLinkToLastBuild: false,
                keepAll: true,
                reportDir: 'tests/_reports/coverage',
                reportFiles: 'index.html',
                reportName: 'Coverage'
            ]
        }

        failure {
            echo 'The Pipeline failed :('
        }
    }
    
}