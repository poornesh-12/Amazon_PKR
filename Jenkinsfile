pipeline {
    agent any


    parameters {
        choice(name: 'DEPLOY_ENV', choices: ['dev', 'qa', 'prod'], description: 'Choose environment')
    }

    environment {
        SONAR_CRED = "sonar-token"
        TOMCAT_CRED = "tomcat-creds"
        WAR_BACKUP_DIR = "backup"
    }

   

    stages {

         stage('Select Environment') {
            steps {
                script {
                    def BRANCH = env.BRANCH_NAME.toLowerCase()

                    DEPLOY_ENV =
                        (BRANCH.startsWith('dev'))  ? 'dev'  :
                        (BRANCH.startsWith('qa'))   ? 'qa'   :
                        (BRANCH.startsWith('prod') || BRANCH == 'main') ? 'prod' :
                                                                         'dev'

                    echo "Branch Name: ${BRANCH}"
                    echo "Selected Environment: ${DEPLOY_ENV}"
                }
            }
        }

        stage('Compile') {
            steps {
                dir('Amazon'){
                    sh 'mvn clean compile -B'
                }
                 // batch mode, jenkins will not interactive question during build (do this or not ...)
            }
        }

        stage('Unit Test') {
            steps {
                dir('Amazon'){
                sh 'mvn test -B'
                }
                junit '**/target/surefire-reports/*.xml' // tells jenkins to go this path and read status failed or passed so that jenkins can display status of test case
            }
        }


        stage('Package WAR') {
            steps {
                dir('Amazon'){
                sh 'mvn -DskipTests package -B'
                }
            }
        }

        stage('Backup WAR') {
            steps {
                sh "mkdir -p ${WAR_BACKUP_DIR}"
                sh "cp Amazon/Amazon-Web/target/*.war ${WAR_BACKUP_DIR}/amazon_${BUILD_NUMBER}.war"
            }
        }

        stage('Deploy to Tomcat') {
            steps {
                script {
                    
                    def BRANCH = env.BRANCH_NAME.toLowerCase()

                    
                    def HOST = (params.DEPLOY_ENV == 'dev')  ? "192.168.127.128:8085"  :
                               (params.DEPLOY_ENV == 'qa')   ? "192.168.127.128:8089"   :
                                                                 "192.168.127.128:8083"

                    withCredentials([usernamePassword(credentialsId: 'local tomcat', usernameVariable: 'USER', passwordVariable: 'PASS')]) { //USER , PASS is jenkins variables we do not need to replace with ACTUAL values

                        sh """
                            echo "Undeploy old version"
                            curl -s -u $USER:$PASS "http://$HOST/manager/text/undeploy?path=/amazon" || true

                            echo "Deploying new WAR"
                            curl -s -u $USER:$PASS --upload-file Amazon/Amazon-Web/target/*.war \
                               "http://$HOST/manager/text/deploy?path=/amazon&update=true"
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: "${WAR_BACKUP_DIR}/*.war", onlyIfSuccessful: false
        }
    }
}
