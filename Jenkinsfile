pipeline {
  agent any
  stages {
    stage('Checkout and Compile') {

      steps {
        tool(name: 'Maven-3.3.9', type: 'maven')
        tool(name: 'SonarQube Scanner 2.6.1', type: 'hudson.plugins.sonar.SonarRunnerInstallation')
        echo 'Checkout and Compile'
        svn(url: 'https://svn.apache.org/repos/asf/excalibur/trunk/', changelog: true, poll: true)
        sh 'mvn clean compile'     
      }
    }
    stage('QA and Tests') {
      steps {
        echo 'QA and Test'
        sh 'mvn package'       
      }
    }
    stage('Deploy') {
      steps {
        echo 'Deploy'
      }
    }
  }
}##
