pipeline {
  agent any
  stages {
    stage ("Build") {
      steps{
        echo "runnung the build automation"
        sh "./gradlew build --no-daemon"
        archiveArtifacts artifacts: 'dist/trainSchedule.zip'
        echo "build completed
      }
    }
  }
}
