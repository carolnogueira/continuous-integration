pipeline {
  agent any
  stages {
    stage('Checkout and Compile') {
      steps {
        svn(url: 'https://svn.apache.org/repos/asf/excalibur/trunk/', changelog: true, poll: true)
        sh 'mvn clean compile'
      }
    }
    stage('QA and Tests') {
      steps {
        echo 'Hello'
        sh 'mvn package'
      }
    }
    stage('Deploy') {
      steps {
        echo 'Hello'
      }
    }
  }
}