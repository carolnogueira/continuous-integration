pipeline {
  agent any
  stages {
    stage('Checkout and Compile') {
      steps {
        svn(url: 'https://svn.apache.org/repos/asf/excalibur/trunk/', changelog: true, poll: true)
      }
    }
    stage('QA and Tests') {
      steps {
        echo 'Hello'
      }
    }
    stage('Deploy') {
      steps {
        echo 'Hello'
      }
    }
  }
}