pipeline{
 parameters {
            string(name: 'SKIP_DEPLOY', defaultValue: "false", description: 'When set to true, the pipeline skips deploying to target servers')
            string(name: 'FAILURE_RESULTS', defaultValue: "", description: 'Post-deployment test failure reuslts')            
            string(name: 'SERVICE_NAME', defaultValue: "smartService", description: 'Name of the service')
            string(name: 'DEPLOY_ENV', defaultValue: "dev", description: 'dev for DEV servers, qa for QA servers')
            string(name: 'RUNTIME_PROFILE', defaultValue: "bmscluster", description: 'Spring profile to use while deploying to target servers')
            string(name: 'DEV_CLUSTER', defaultValue: "bmscluster", description: 'DEV Spring profile')
            string(name: 'QA_CLUSTER', defaultValue: "bmscluster", description: 'QA Spring  profile')              
            string(name: 'PORT1', defaultValue: "3800", description: 'Service port number 1')
            string(name: 'PORT2', defaultValue: "3900", description: 'Service port number 2')
            string(name: 'MAVEN_ACTION', defaultValue: "", description: 'Maven action to perform')
            string(name: 'ALERT_EMAILID', defaultValue: 'axs00yj@fpl.com' )
        }
        
 agent none
 stages{
 stage ('Clone-Repo') {
        agent{label 'master'}
        steps{
            script{                    
                    git url: 'http://bitbucket.com/somerepo.git', credentialsId: 'bff6047c-e44f-4215-8a1b-eacd568fccb2'
            }
             
        }
        post
            {
                failure {
                    emailext attachLog: true,
                    body: "Something is wrong with Clone-Repo step, please check Build ID : ${currentBuild.currentResult}: ${BUILD_URL} :${env.BUILD_ID}" ,
                    compressLog: false, 
                    to: env.ALERT_EMAILID,
                    subject: "Build Failure Notification: ${JOB_NAME}-Build# ${BUILD_NUMBER} ${currentBuild.currentResult}"
                    //from: 'JenkinsBuild'                    
                       }
            }       
    }
 
 stage ('Code-Analysis'){
        agent{label 'master'}
            steps{
                script{
                def scannerHome = tool 'sonar scanner'                
                withSonarQubeEnv('sonarqube'){
                    sh "$scannerHome/bin/sonar-scanner -Dsonar.projectKey=smartoutage -Dsonar.projectName=smartoutage -Dsonar.projectVersion=1.0 -Dsonar.sources=. -Dsonar.sourceEncoding=utf-8 -Dsonar.scm.provider=git -Dsonar.scm.disabled=true -Dsonar.java.binaries=."
                  }
                }
            }
        post
           {
               failure {
                    emailext attachLog: true,
                    body: "Something is wrong with Code-Analysis step, please check Build ID : ${currentBuild.currentResult}: ${BUILD_URL} ,${env.BUILD_ID}", 
                    compressLog: false, 
                    to: env.ALERT_EMAILID,
                    subject: "Build Failure Notification: ${JOB_NAME}-Build# ${BUILD_NUMBER} ${currentBuild.currentResult}"
                    //from: 'JenkinsBuild'                    
               }                          
            }
    }
 
 stage ('Maven-Actions') {
        agent{label 'master'}
        steps {
           script{
                    if(env.BRANCH_NAME == 'master'){
                        echo "***MASTER BRANCH - validate only***"
                        env.MAVEN_ACTION = "validate_only"
                    }
                    else if(env.BRANCH_NAME.contains("feature/") && isSnapshotBuild() == true){
                        echo "***FEATURE BRANCH - snapshot artifact***"
                        env.MAVEN_ACTION = "upload_snapshot"
                    }
                    else if(env.BRANCH_NAME.contains("bugfix/") && isSnapshotBuild() == true){
                        echo "***BUGFIX BRANCH - snapshot artifact***"
                        env.MAVEN_ACTION = "upload_snapshot"
                    }
                    else if(env.BRANCH_NAME.contains("release/") && isSnapshotBuild() == true){
                        echo "***RELEASE BRANCH - snapshot artifact***"
                        env.MAVEN_ACTION = "upload_snapshot"
                    }
                    else if(env.BRANCH_NAME.contains("release/") && isSnapshotBuild() == false){
                        echo "***RELEASE BRANCH - release artifact***"
                        env.MAVEN_ACTION = "upload_release"
                    }
                }                 
           script{  
               if( env.MAVEN_ACTION == 'upload_snapshot' || env.MAVEN_ACTION == 'upload_release' ){    
                    withMaven(maven : 'mvn') {
                    sh "mvn -f pom.xml clean deploy -DskipTests=false"
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'impl/target/site/jacoco-ut', reportFiles: 'index.html',
                    reportName: 'UnitTestCodeCoverageReport', reportTitles: ''])
                
                 }
               }    
               else if( env.MAVEN_ACTION == 'validate_only' ){    
                    withMaven(maven : 'mvn') {                    
                    sh "mvn -f pom.xml clean validate -DskipTests=false"                                 
                 }
               }       
           }
        }
        post
            {
                failure {
                    emailext attachLog: true,
                    body: "Something is wrong with Maven-Actions step, please check Build ID : ${currentBuild.currentResult}: ${BUILD_URL} :${env.BUILD_ID}" ,
                    compressLog: false, 
                    to: env.ALERT_EMAILID,
                    subject: "Build Failure Notification: ${JOB_NAME}-Build# ${BUILD_NUMBER} ${currentBuild.currentResult}"
                    //from: 'JenkinsBuild'
                }
            }      
  } 

  stage ('Deployment-Configuration') {
    agent{label 'master'}
    steps{
            script{
                
                jenkinsWS = "${env.WORKSPACE}"                                                                                                                                                      
                logPath = "/opt/data/logs/smartoutage/smart_service"
                servicePath =  "/opt/data/pdim_services"  
                memOptions = "-Xmx5120m -Xms5120m"        
                runAsUser = "pdimuser"   
                runAsGroup = "pdimuser"                                                           
                                                                 
                globalServiceName = "serviceName"
                serviceName1 = "${env.SERVICE_NAME}1-${env.PORT1}"
                serviceName2 = "${env.SERVICE_NAME}2-${env.PORT2}" 
                portList = [1:"${env.PORT1}", 2:"${env.PORT2}"]
                devServerList = ["goxsd5172", "goxsd5173", "goxsd5174"]
                qaServerList = ["goxsd5189", "goxsd5190", "goxsd5191"]
                finalServerList = devServerList
                
                script{                    
                    if(env.BRANCH_NAME.contains("release/") && isSnapshotBuild() == false){
                        env.DEPLOY_ENV = "qa" 
                        env.RUNTIME_PROFILE = env.QA_CLUSTER
                        finalServerList = qaServerList

                    }
                    else if (env.MAVEN_ACTION == "upload_snapshot"){
                        env.DEPLOY_ENV = "dev" 
                        env.RUNTIME_PROFILE = env.DEV_CLUSTER
                        finalServerList = devServerList
                    }
                }                  
            }
          }
  }
     

 stage('Service-Deployment'){
          agent{label 'master'}           
          steps{           
            
            script {   
              
                    if(env.SKIP_DEPLOY == "false" && env.BRANCH_NAME != 'master'){

                      echo "Deploying to ${env.DEPLOY_ENV}"                      
                      sh "sed -i 's/`whoami`/$runAsUser/g' $jenkinsWS/spring-boot-jar/target/service_controller.sh"  
                      deployService(finalServerList)                                               
                     
                    }
              }
          }

          post
          {
            failure {              
              emailext attachLog: true,
              body: "Something is wrong with Service-Deployment step, please check Build ID : ${currentBuild.currentResult}: ${BUILD_URL} :${env.BUILD_ID}\n${env.FAILURE_RESULTS}" ,
              compressLog: false, 
              to: env.ALERT_EMAILID,
              subject: "Build Failure Notification: ${JOB_NAME}-Build# ${BUILD_NUMBER} ${currentBuild.currentResult}"
              //from: 'JenkinsBuild'                    
            }
          }
    }   

     stage('Post-Deployment Test'){
          agent{label 'master'}
          steps{           
            
            script {                       
                    if(env.SKIP_DEPLOY == "false"){                                                                                     
                      sh "sleep 10"
                      testService(finalServerList)                                    
                   }                           
              }                  
          }    
          post
          {
            failure {
              println("${env.FAILURE_RESULTS}")
              emailext attachLog: true,
              body: "Something is wrong with Post-Deployment Test step, please check Build ID : ${currentBuild.currentResult}: ${BUILD_URL} :${env.BUILD_ID}\n${env.FAILURE_RESULTS}" ,
              compressLog: false, 
              to: env.ALERT_EMAILID,
              subject: "Build Failure Notification: ${JOB_NAME}-Build# ${BUILD_NUMBER} ${currentBuild.currentResult}"
              //from: 'JenkinsBuild'                    
            }                                    
          }         
    }
  }
 }

 def deployService(serverList){
      serverList.each{server ->
        portList.each{port ->
          evaluate("${globalServiceName} = serviceName$port.key")
          portNumber = port.value                                                      
          withCredentials([usernamePassword(credentialsId: 'zzzjenkins',
             usernameVariable: 'username', passwordVariable: 'pw')]) {                 

               def scripttext = ""

               sh """sshpass -p ${pw} ssh -o StrictHostKeyChecking=no ${username}@$server mkdir -p $servicePath/$serviceName"""
               sh """sshpass -p ${pw} ssh -o StrictHostKeyChecking=no ${username}@$server sudo chmod 775 -R $servicePath/$serviceName"""
               sh """sshpass -p ${pw} ssh -o StrictHostKeyChecking=no ${username}@$server mkdir -p $logPath"""
               sh """sshpass -p ${pw} ssh -o StrictHostKeyChecking=no ${username}@$server sudo chmod 775 -R $logPath"""     
               sh """sshpass -p ${pw} ssh -o StrictHostKeyChecking=no ${username}@$server sudo chown -R $runAsUser:$runAsGroup $logPath"""                                
               sh """sshpass -p ${pw} scp -o StrictHostKeyChecking=no $jenkinsWS/spring-boot-jar/target/systemd.conf.template ${username}@$server:$servicePath/$serviceName/systemd.conf.template"""
               sh """sshpass -p ${pw} scp -o StrictHostKeyChecking=no $jenkinsWS/spring-boot-jar/target/service_controller.sh ${username}@$server:$servicePath/$serviceName/"""                                                              
               sh """sshpass -p ${pw} scp -o StrictHostKeyChecking=no $jenkinsWS/spring-boot-jar/target/*.jar ${username}@$server:$servicePath/$serviceName/"""                                                                         
               sh """sshpass -p ${pw} ssh -o StrictHostKeyChecking=no ${username}@$server $servicePath/$serviceName/service_controller.sh -a reinstall-service -n $serviceName"""                             
               sh """sshpass -p ${pw} ssh -o StrictHostKeyChecking=no ${username}@$server $servicePath/$serviceName/service_controller.sh -a configure -o \\"$memOptions\\" -n $serviceName -e ${env.RUNTIME_PROFILE} -p $portNumber -l $logPath -s $servicePath/$serviceName/stats"""
               sh """sshpass -p ${pw} scp -o StrictHostKeyChecking=no $jenkinsWS/spring-boot-jar/*.yml ${username}@$server:$servicePath/$serviceName/current.$serviceName/"""
               sh """sshpass -p ${pw} ssh -o StrictHostKeyChecking=no ${username}@$server sudo chmod 775 -R $servicePath/$serviceName"""
               sh """sshpass -p ${pw} ssh -o StrictHostKeyChecking=no ${username}@$server sudo chown -R $runAsUser:$runAsGroup $servicePath/$serviceName""" 
               sh """sshpass -p ${pw} ssh -o StrictHostKeyChecking=no ${username}@$server $servicePath/$serviceName/service_controller.sh -a restart -n $serviceName"""                             
               sh """sshpass -p ${pw} ssh -o StrictHostKeyChecking=no ${username}@$server sudo chmod 775 -R $servicePath/$serviceName"""
               sh """sshpass -p ${pw} ssh -o StrictHostKeyChecking=no ${username}@$server sudo chown -R $runAsUser:$runAsGroup $servicePath/$serviceName""" 
               sh """sshpass -p ${pw} ssh -o StrictHostKeyChecking=no ${username}@$server sudo chmod 775 -R $logPath"""
               sh """sshpass -p ${pw} ssh -o StrictHostKeyChecking=no ${username}@$server sudo chown -R $runAsUser:$runAsGroup $logPath"""                                 
        }
      }
    }
 }


 def testService(serverList){      
      serverList.each{server ->
        portList.each{port ->
          evaluate("${globalServiceName} = serviceName$port.key")
          portNumber = port.value                                                                        
          def httpResponse           
          try{
            sh "sleep 5"
            httpResponse = sh (returnStdout: true, script: """curl -s -o /dev/null -w "%{http_code}" --header "FPLSO-appid: sotest" --header "FPLSO-slid: jenkinsDeployer" http://$server:$portNumber/v1/ticketstatus/premise/0 """)          
            if(httpResponse.contains("200")){
                 println("""THE SERVICE IS UP ON $server:$portNumber !""")               
            }
            else{                            
                 currentBuild.result = 'FAILURE'
                 if(env.FAILURE_RESULTS == ""){
                      env.FAILURE_RESULTS = """EXCEPTIONS ENCOUNTERED :\nTHE SERVICE IS NOT UP ON $server:$portNumber !\n"""
                 }
                 else{
                      env.FAILURE_RESULTS = env.FAILURE_RESULTS + """THE SERVICE IS NOT UP ON $server:$portNumber !\n"""
                 }
               }
            }
          catch(error){  
               currentBuild.result = 'FAILURE'
               if(env.FAILURE_RESULTS == ""){
                    env.FAILURE_RESULTS = """EXCEPTIONS ENCOUNTERED :\nTHE SERVICE IS NOT UP ON $server:$portNumber !\n"""
               }
               else{
                    env.FAILURE_RESULTS = env.FAILURE_RESULTS + """THE SERVICE IS NOT UP ON $server:$portNumber !\n"""
               }
          }
      }
    }
 }

 def getReleaseVersion() {
    return (readFile('pom.xml') =~ '<version>(.+)-SNAPSHOT</version>')[0][1]
 }

 def isSnapshotBuild() {
    return (readFile('pom.xml').contains("-SNAPSHOT</version>"))
 }
