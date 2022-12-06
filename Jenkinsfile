pipeline {
    agent any
    
    tools {
        maven "mvn"
    }
    
    stages {
        stage("gitpull") {
            steps {
                git branch: 'main', credentialsId: 'github', url: 'git@github.com:sathishbob/spring-petclinic.git'
            }
        }
        stage("gitleak") {
            steps {
                sh '''sudo docker run -v /var/lib/jenkins/workspace/devsecopspipeline:/path zricethezav/gitleaks:latest detect -v --no-git --source="/path"
'''
            }
        }
        stage("SCA") {
            steps {
                sh '''DC_VERSION="latest"
                      DC_DIRECTORY=$HOME/OWASP-Dependency-Check
                      DC_PROJECT="dependency-check scan: $(pwd)"
                      DATA_DIRECTORY="$DC_DIRECTORY/data"
                      CACHE_DIRECTORY="$DC_DIRECTORY/data/cache"

                      if [ ! -d "$DATA_DIRECTORY" ]; then
                         echo "Initially creating persistent directory: $DATA_DIRECTORY"
                         mkdir -p "$DATA_DIRECTORY"
                      fi
                      if [ ! -d "$CACHE_DIRECTORY" ]; then
                         echo "Initially creating persistent directory: $CACHE_DIRECTORY"
                         mkdir -p "$CACHE_DIRECTORY"
                      fi

                      mkdir odc-reports || true
                      chown jenkins:jenkins odc-reports
                      # Make sure we are using the latest version
                      sudo docker pull owasp/dependency-check:$DC_VERSION

                      sudo docker run --rm \\
                      -e user=$USER \\
                      -u $(id -u ${USER}):$(id -g ${USER}) \\
                      --volume $(pwd):/src:z \\
                      --volume "$DATA_DIRECTORY":/usr/share/dependency-check/data:z \\
                      --volume $(pwd)/odc-reports:/report:z \\
                      owasp/dependency-check:$DC_VERSION \\
                      --scan /src \\
                      --format "ALL" \\
                      --project "$DC_PROJECT" \\
                      --out /report
                      # Use suppression like this: (where /src == $pwd)
                      # --suppression "/src/security/dependency-check-suppression.xml"
                    '''
            }
            post {
                success {
                    dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
                }
            }
        }
        stage("Build Source") {
            steps {
               sh '''rm -rf odc-reports
                     rm -rf .scannerwork
                     rm -rf out''' 
                sh " mvn package"     
            }
        }
        stage("SAST") {
            steps {
                script {
                    scannerHome = tool 'sonar';
                    withSonarQubeEnv('sonar') {
                    sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectName=javaapplication -Dsonar.projectKey=java:findbugs -Dsonar.projectVersion=1.0 -Dsonar.projectBaseDir=$WORKSPACE -Dsonar.sources=$WORKSPACE -Dsonar.java.libraries=$WORKSPACE -Dsonar.java.binaries=$WORKSPACE"
                    sh "sleep 20"
                    }
                }
            }
        }
        stage("SAST Qualitiy Gate") {
            steps {
                script {
                    timeout(time: 1, unit: 'HOURS') {
                        waitForQualityGate abortPipeline: true
                    }
                }
            }
        }
        stage("Conatiner Build") {
            steps {
                sh '''sudo docker build -t sathishbob/javapetclinic .
                sudo docker push sathishbob/javapetclinic
                echo sathishbob/javapetclinic > anchore_images'''
            }
        }
        stage("Containet Scan") {
            steps {
                anchore bailOnFail: false, name: 'anchore_images'
            }
        }
        stage("Deployment Approval") {
            steps {
                input "Please approve to proceed with deployment"
            }
        }
        stage("Dev Deployment") {
            steps {
                sh "sudo docker run -d -p 8081:8080 sathishbob/javapetclinic"
            }
        }
        stage("DAST") {
            steps {
                sh '''mkdir -p $WORKSPACE/out
                    chmod 777 $WORKSPACE/out
                    rm -f $WORKSPACE/out/*.*
                    sudo docker run --rm -v $WORKSPACE/out:/zap/wrk/:rw -t owasp/zap2docker-live zap-baseline.py -t http://44.211.50.191:8081/ -m 15 -d -r ${JOB_NAME}_Dev_ZAP_VULNERABILITY_REPORT_${BUILD_ID}.html -x ${JOB_NAME}_Dev_ZAP_VULNERABILITY_REPORT_${BUILD_ID}.xml || true
                    '''
            }
            post {
                success {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'out/', reportFiles: '*.html', reportName: 'DASTReport', reportTitles: 'ZAP_report.html', useWrapperFileDirectly: true])
                }
            }
        }
    }
}
