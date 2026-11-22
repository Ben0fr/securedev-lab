pipeline {
  agent any
  stages {
    stage('Construccion') {
      steps { sh 'docker compose -p securedev-estudiante build' }
    }
    stage('Pruebas') {
      steps { sh 'echo "Pruebas pendientes"' }
    }
    stage('Despliegue') {
      steps { sh 'docker compose -p securedev-estudiante up -d --build' }
    }
  }
}
