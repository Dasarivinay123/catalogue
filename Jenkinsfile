pipeline {
    agent {
        node {
            label 'ROBOSHOP'
        }
    }

    environment {
        APP_VERSION = ''
        acc_id = '039077527503'
        project = 'roboshop'
        component = 'catalogue'
    }

    options {
        disableConcurrentBuilds()
        timeout(time: 15, unit: 'MINUTES')
    }

    stages {
        stage('Read version') {
            steps {
                script {
                    def packageJson = readJSON file: 'package.json'
                    env.APP_VERSION = packageJson.version
                    echo "The application version is: ${env.APP_VERSION}"
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Unit tests') {
            steps {
                sh 'npm test'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh "${tool 'sonar-8'}/bin/sonar-scanner"
                }
            }
        }

        stage('SonarQube Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    script {
                        def qg = waitForQualityGate()

                        if (qg.status != 'OK') {
                            error "Pipeline stopped: SonarQube Quality Gate status is ${qg.status}"
                        }

                        echo 'SonarQube Quality Gate passed.'
                    }
                }
            }
        }

        stage('Check Dependabot Alerts') {
            steps {
                withCredentials([
                    string(credentialsId: 'github-token', variable: 'GH_TOKEN')
                ]) {
                    sh '''
                        set -eu

                        REPO="Dasarivinay123/catalogue"

                        echo "Checking open Dependabot alerts for ${REPO}"

                        curl --fail --silent --show-error --location \
                            -H "Accept: application/vnd.github+json" \
                            -H "Authorization: Bearer ${GH_TOKEN}" \
                            -H "X-GitHub-Api-Version: 2026-03-10" \
                            "https://api.github.com/repos/${REPO}/dependabot/alerts?state=open&per_page=100" \
                            -o alerts.json

                        echo "---- Open Dependabot Alerts ----"

                        jq -e 'type == "array"' alerts.json > /dev/null

                        jq -r '.[] | "\(.number)\t\(.security_vulnerability.severity)\t\(.dependency.package.name)\t\(.security_advisory.ghsa_id)"' alerts.json

                        HIGH_CRITICAL_COUNT=$(jq '[.[] | select(
                            .security_vulnerability.severity == "high" or
                            .security_vulnerability.severity == "critical"
                        )] | length' alerts.json)

                        echo "High/Critical alert count: ${HIGH_CRITICAL_COUNT}"

                        if [ "${HIGH_CRITICAL_COUNT}" -gt 0 ]; then
                            echo "ERROR: Found ${HIGH_CRITICAL_COUNT} open high/critical Dependabot alert(s)."
                            echo "Docker build and deployment will not proceed."
                            exit 1
                        else
                            echo "No open high/critical Dependabot alerts found."
                        fi
                    '''
                }
            }
        }

        stage('Docker Build') {
            steps {
                withAWS(credentials: 'aws_creds', region: 'us-east-1') {
                    sh '''
                        set -eu

                        IMAGE="${acc_id}.dkr.ecr.us-east-1.amazonaws.com/${project}/${component}:${APP_VERSION}"

                        aws ecr get-login-password --region us-east-1 |
                            docker login \
                                --username AWS \
                                --password-stdin "${acc_id}.dkr.ecr.us-east-1.amazonaws.com"

                        docker build -t "${IMAGE}" .
                        docker push "${IMAGE}"
                    '''
                }
            }
        }

        stage('Deploy') {
            when {
                expression { return params.DEPLOY }
            }

            input {
                message 'Should we continue?'
                ok 'Yes, we should.'
                submitter 'alice,bob'

                parameters {
                    string(
                        name: 'PERSON',
                        defaultValue: 'Mr Jenkins',
                        description: 'Who should I say hello to?'
                    )
                }
            }

            steps {
                echo 'Deploying....'
            }
        }
    }

    post {
        always {
            echo 'I will always say Hello again!'
        }

        success {
            echo 'I will run when it is successful!'
        }

        failure {
            echo 'I will run when it is failed!'
        }
    }
}