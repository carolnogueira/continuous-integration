pipeline {
  agent any
  stages {
    stage('Checkout and Compile') {
      steps {
        svn(url: 'https://svn.apache.org/repos/asf/excalibur/trunk/', changelog: true, poll: true)
        sh 'mvn clean compile'
        tool(name: 'Maven-3.3.9', type: 'Maven')
        echo 'Checkout and Compile'
      }
    }
    stage('QA and Tests') {
      steps {
        echo 'QA and Test'
        sh 'mvn package'
        tool(name: 'SonarQube Scanner 2.6.1', type: 'SonarQube')
      }
    }
    stage('Deploy') {
      steps {
        echo 'Deploy'
      }
    }
  }
}