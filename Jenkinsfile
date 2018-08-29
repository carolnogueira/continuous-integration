/* This Jenkinsfile is intended to run on MPDFT and may fail anywhere else.
* It makes assumptions about plugins being installed, labels mapping to nodes that can build what is needed, etc.
*/

def SONAR_URL = "http://sonar"
def SVN_REPO = "https//svn.mycompany.com/myproject"
def POM_PATH = "pom.xml"
def REMOTE_SERVER = "mydeploymachine"

node {
    // tools
    def MAVEN_HOME = tool "Maven 3.3.9"
    def SONAR_HOME = tool "SonarQube Scanner 3.0.0.702"
    def SONAR_JAVA_HOME = tool "Java 8_121"
    def JAVA_HOME = tool "Java 8_121"
    
    def canSkipTests = false
    
    stage('Checkout and Compile') {
        println 'Causa da geração da build: ' + build_cause()

        env.JAVA_HOME="${JAVA_HOME}"
        env.PATH="${env.JAVA_HOME}/bin:${env.PATH}"

        try {
            timeout(time: 20, unit: 'SECONDS') {
                input message: 'Deseja pular os testes?', ok: 'Sim'
                canSkipTests = true
            }
        } catch (err) {            
            errorAnalysis(err, SVN_REPO, 0)
        }
        
        echo 'Realizando checkout do projeto...'
        svn SVN_REPO    
        
        try {
            echo 'Compilando o projeto...'
            sh "${MAVEN_HOME}/bin/mvn -f ${POM_PATH} clean compile -Dmaven.test.skip=true -Pdesenvolvimento"
        
        } catch(err) {
            errorAnalysis(err, SVN_REPO, 1)
            error "Falha na compilação, por favor leia os logs.."       
        }
    }

   if (!canSkipTests) {             
        stage('QA and Tests') {
            try {
                echo 'Executando testes...'
                sh "${MAVEN_HOME}/bin/mvn -f ${POM_PATH} clean package org.jacoco:jacoco-maven-plugin:prepare-agent -Dmaven.test.skip=false -Pdesenvolvimento -DpjeDbUnitHost=selenium -Djasmine.serverHostname=\$(hostname)"
                
            } catch(err) {
                errorAnalysis(err, SVN_REPO, 3)
            //  error "Falha nos testes, por favor leia os logs.."       
            } finally {
                step([$class: 'JUnitResultArchiver', allowEmptyResults: true, testResults: '**/target/*-reports/TEST-*.xml'])
                publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true, reportDir: 'target/br/mp/mpdft/pjempdft/estoria', reportFiles: 'Index.html', reportName: 'Concordion'])
                jacoco()
            }
            
            try {
                echo 'Análisando a qualidade'
                def file = getGroupIdFromPom(POM_PATH) + ":" + getArtifactIdFromPom(POM_PATH)
                
                withSonarQubeEnv('Sonar 6.1') {
                    withEnv(["JAVA_HOME=${SONAR_JAVA_HOME}"]) {
                        sh "${MAVEN_HOME}/bin/mvn -f ${POM_PATH} compile verify org.sonarsource.scanner.maven:sonar-maven-plugin:3.1.1:sonar -Dsonar.projectKey=${file} -Dmaven.test.skip=true -Pdesenvolvimento"                 
                    }
                }
                sh 'sleep 4s'
            
                def response = httpRequest "${SONAR_URL}/api/qualitygates/project_status?projectKey=${file}"
                def slurper = new groovy.json.JsonSlurper()
                def result = slurper.parseText(response.content)
                
                if (result.projectStatus.status == "ERROR") {
                    error "Falha na qualidade do código, por favor leia os logs.."
                }
            } catch(err) {
                errorAnalysis(err, SVN_REPO, 2)
              
            } finally {
                findbugs canRunOnFailed: true, canComputeNew: false, defaultEncoding: '', excludePattern: '', healthy: '', includePattern: '', pattern: 'target/sonar/findbugs-result.xml', unHealthy: ''
            }
        }      
    }

   if ( build_cause() instanceof  hudson.model.Cause$UserIdCause ) {
        stage('Deploy') {
            echo 'Empacontando o projeto...'
            sh "${MAVEN_HOME}/bin/mvn -f ${POM_PATH} clean package -Dmaven.test.skip=true -Pdesenvolvimento"
            archiveArtifacts allowEmptyArchive: true, artifacts: 'target/*.war', onlyIfSuccessful: true

            echo 'Deploy'
            withCredentials([usernamePassword(credentialsId: 'continuous-delivery', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')]) {                
                def file = getArtifactIdFromPom(POM_PATH) + "-" + getVersionFromPom(POM_PATH) + '-desenvolvimento.war'

                echo "Sending '${file}' over ssh.."    
                sh "sshpass -p '${PASSWORD}' scp target/'${file}' '${USERNAME}'@'${REMOTE_SERVER}':/home/MPDFT/ci-deploy"
                sh 'sleep 10s'
                sh "sshpass -p '${PASSWORD}' ssh '${USERNAME}'@'${REMOTE_SERVER}' \"sudo /opt/wildfly9_1/bin/jboss-cli.sh --connect --controller=localhost:19990 --command='undeploy pjempdft-*.war' || true\""				
                sh "sshpass -p '${PASSWORD}' ssh '${USERNAME}'@'${REMOTE_SERVER}' \"sudo /opt/wildfly9_1/bin/jboss-cli.sh --connect --controller=localhost:19990 --command='deploy --force /home/MPDFT/ci-deploy/${file}'\""
            }              
        }
    }
}

def getGroupIdFromPom(pom) {
  def matcher = readFile(pom) =~ '<groupId>(.+)</groupId>'
  matcher ? matcher[0][1] : null
 }

def getVersionFromPom(pom) {
  def matcher = readFile(pom) =~ '<version>(.+)</version>'
  matcher ? matcher[0][1] : null
 }

def getArtifactIdFromPom(pom) {
  def matcher = readFile(pom) =~ '<artifactId>(.+)</artifactId>'
  matcher ? matcher[0][1] : null
}

@NonCPS
def build_cause() {
    return currentBuild.rawBuild.getCauses()[0]
}

def getLastCommitAuthor(repository) {
    RESULT = sh (
        script: "svn log -q -v --limit 1  '${repository}' | grep \"^r\" | awk '{print \$3}'",
        returnStdout: true
    ).trim()
    return RESULT
}

def getLastCommitMessage(repository) {
    RESULT = sh (
        script: "svn log -q -v --xml --with-all-revprops --limit 1 '${repository}' | grep msg | sed -e \"s/<msg>\\([^<]*\\)<\\/msg>/\\1/g\"",
        returnStdout: true
    ).trim()
    return RESULT
}

def sendNotification(mailMessage, mailSubject) {
    println "Notificação enviada: ${mailSubject} [${env.BUILD_URL}]"
    // mail body: mailMessage,
    // from: 'jenkins-noreply@mpdft.gov.br',
    // replyTo: 'jenkins-noreply@mpdft.gov.br',
    // subject: mailSubject,
    // to: 'nudes1-todos@mpdft.mp.br, nudes2-todos@mpdft.mp.br'
    
   // slackSend (color: '#FF0000', message: "${mailSubject} [${env.BUILD_URL}]")
}

def sendNotificationBuildFailedQA(repository) {
    COMMITTER_LOGIN = getLastCommitAuthor(repository)
    COMMITTER_MESSAGE = getLastCommitMessage(repository)
      
    def messageBuildFailed = """
    Olá pessoal, 
            
        Falhou a análise de qualidade de código do build de número ${env.BUILD_NUMBER} do job ${env.JOB_NAME}, acho que alguma coisa no último commit não deu certo. 
        Você poderia acessar o link ${env.BUILD_URL} e verificar o motivo da falha? 

        Informações sobre o commit são:
            Autor: ${COMMITTER_LOGIN}
            Mensagem: ${COMMITTER_MESSAGE}
        
        Agradeço desde já.
         
    Jenkins"""
    sendNotification(messageBuildFailed, "[FALHA NA QUALIDADE DO CÓDIGO]")        
}

def sendNotificationBuildFailedTests(repository) {
    COMMITTER_LOGIN = getLastCommitAuthor(repository)
    COMMITTER_MESSAGE = getLastCommitMessage(repository)
    
    def messageBuildFailed = """
    Olá pessoal, 
            
        Os testes automátizados do build de número ${env.BUILD_NUMBER} do job ${env.JOB_NAME} falharam, acho que alguma coisa no último commit não deu certo. 
        Você poderia acessar o link ${env.BUILD_URL} e verificar o motivo da falha? 

        Informações sobre o commit são:
            Autor: ${COMMITTER_LOGIN}
            Mensagem: ${COMMITTER_MESSAGE}
        
        Agradeço desde já.
         
    Jenkins"""
    sendNotification(messageBuildFailed, "[FALHA NOS TESTES]")        
}

def sendNotificationCompileProblem(repository) {
    COMMITTER_LOGIN = getLastCommitAuthor(repository)
    COMMITTER_MESSAGE = getLastCommitMessage(repository)
    
    def messageBuildFailed = """
    Gente, 
            
        Ocorreu um erro de compilação com o build de número ${env.BUILD_NUMBER} do job ${env.JOB_NAME}, acho que alguma coisa no último commit não deu certo. 
        Você poderia acessar o link ${env.BUILD_URL} e verificar o motivo da falha? 

        Informações sobre o commit são:
            Autor: ${COMMITTER_LOGIN}
            Mensagem: ${COMMITTER_MESSAGE}
        
        Agradeço desde já.
         
    Jenkins"""
    sendNotification(messageBuildFailed, "[ERRO DE COMPILAÇÃO]")        
}

@NonCPS
def errorAnalysis(err, repository, reason) {
    echo "Error: ${err}; Message: ${err.getMessage()}"
    
    if (err.getMessage() == null) {
        return;
    }
    
    if ( err.getMessage().contains("script returned exit code 143")) {
        currentBuild.result='UNSTABLE'
        throw err
    }
        
    if ( reason == 1 ) {
        currentBuild.result = 'FAILURE'
        sendNotificationCompileProblem(repository)
        
    } else if ( reason == 2 ) {
        //currentBuild.result='UNSTABLE'
        sendNotificationBuildFailedQA(repository) 
           
    } else if ( reason == 3 ) {   
        currentBuild.result = 'FAILURE'
        sendNotificationBuildFailedTests(repository) 
    } 
}
