def COLOR_MAP = [
    SUCCESS: 'good',
    FAILURE: 'danger',
    UNSTABLE: 'warning',
    ABORTED: '#808080'
]

pipeline {
    agent any

    environment {
        WORKSPACE = "${env.WORKSPACE}"
        NEXUS_CREDENTIAL_ID = 'Nexus-Credential'
    }

    tools {
        maven 'localMaven'
        jdk 'localJdk'
    }

    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
            post {
                success {
                    echo ' now Archiving '
                    archiveArtifacts artifacts: '**/*.war'
                }
            }
        }

        stage('Unit Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Integration Test') {
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }

        stage('Checkstyle Code Analysis') {
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }

        /*stage('SonarQube Inspection') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    withCredentials([string(credentialsId: 'SonarQube-Token', variable: 'SONAR_TOKEN')]) {
                       sh 'printenv | grep SONAR'

                        sh """
                        mvn sonar:sonar \
                        -Dsonar.projectKey=JavaWebApp-Project \
                        -Dsonar.login=$SONAR_TOKEN
                        """
                    }
                }
            }
        }

        stage('SonarQube GateKeeper') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }*/

        stage("Nexus Artifact Uploader") {
            steps {
                nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    nexusUrl: '65.0.89.134:8081',
                    groupId: 'webapp',
                    version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
                    repository: 'maven-project-releases',
                    credentialsId: "${NEXUS_CREDENTIAL_ID}",
                    artifacts: [
                        [artifactId: 'webapp',
                         classifier: '',
                         file: "${WORKSPACE}/webapp/target/webapp.war",
                         type: 'war']
                    ]
                )
            }
        }

        stage('Deploy to Development Env') {
            environment {
                HOSTS = 'dev'
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'Ansible-Credential', passwordVariable: 'PASSWORD', usernameVariable: 'USER_NAME')]) {
                    sh "ansible-playbook -i ${WORKSPACE}/ansible-config/aws_ec2.yaml ${WORKSPACE}/deploy.yaml --extra-vars \"ansible_user=$USER_NAME ansible_password=$PASSWORD hosts=tag_Environment_$HOSTS workspace_path=$WORKSPACE\""
                }
            }
        }

        stage('Deploy to Staging Env') {
            environment {
                HOSTS = 'stage'
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'Ansible-Credential', passwordVariable: 'PASSWORD', usernameVariable: 'USER_NAME')]) {
                    sh "ansible-playbook -i ${WORKSPACE}/ansible-config/aws_ec2.yaml ${WORKSPACE}/deploy.yaml --extra-vars \"ansible_user=$USER_NAME ansible_password=$PASSWORD hosts=tag_Environment_$HOSTS workspace_path=$WORKSPACE\""
                }
            }
        }

        stage('Quality Assurance Approval') {
            steps {
                input('Do you want to proceed?')
            }
        }

        stage('Deploy to Production Env') {
            environment {
                HOSTS = 'prod'
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'Ansible-Credential', passwordVariable: 'PASSWORD', usernameVariable: 'USER_NAME')]) {
                    sh "ansible-playbook -i ${WORKSPACE}/ansible-config/aws_ec2.yaml ${WORKSPACE}/deploy.yaml --extra-vars \"ansible_user=$USER_NAME ansible_password=$PASSWORD hosts=tag_Environment_$HOSTS workspace_path=$WORKSPACE\""
                }
            }
        }
    }

    post {
        always {
            script {
                echo 'Sending Slack Notification...'
                withCredentials([string(credentialsId: 'your-slack-bot-token-id', variable: 'SLACK_BOT_TOKEN')]) {
                    slackSend(
                        botToken: SLACK_BOT_TOKEN,
                        channel: '@your_initial-cicd-pipeline-alerts',
                        color: COLOR_MAP[currentBuild.currentResult] ?: '#CCCCCC',
                        message: "*${currentBuild.currentResult}:* Job '${env.JOB_NAME}' build #${env.BUILD_NUMBER}\nBuild Timestamp: ${env.BUILD_TIMESTAMP}\nWorkspace: ${env.WORKSPACE}\nMore info: ${env.BUILD_URL}"
                    )
                }
            }
        }
    }
}
