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
    stage('DAST - OWASP ZAP') {
      steps {
        script {
          def rc = sh(returnStatus: true, script: '''#!/bin/bash
set -uo pipefail
NETWORK=securedev-estudiante_default
TARGET=http://app:8000
mkdir -p reports/zap

echo ">> Obteniendo sesion del usuario de prueba"
SESSION=$(docker run --rm --network "$NETWORK" curlimages/curl:8.11.1 sh -c "curl -s -i -d username=ana -d password=ana123 $TARGET/login | grep -i '^set-cookie: session=' | head -n 1 | cut -d= -f2- | cut -d';' -f1")
if [ -z "$SESSION" ]; then echo "No se pudo iniciar sesion"; exit 3; fi
echo ">> Sesion obtenida (${#SESSION} caracteres)"

ZAP_OPTS="-config replacer.full_list\\(0\\).description=auth-cookie -config replacer.full_list\\(0\\).enabled=true -config replacer.full_list\\(0\\).matchtype=REQ_HEADER -config replacer.full_list\\(0\\).matchstr=Cookie -config replacer.full_list\\(0\\).regex=false -config replacer.full_list\\(0\\).replacement=session=$SESSION -config globalexcludeurl.url_list.url\\(0\\).regex=.*/logout.* -config globalexcludeurl.url_list.url\\(0\\).enabled=true"

CID=$(docker create --user root --network "$NETWORK" zaproxy/zap-stable zap-full-scan.py -t "$TARGET" -m 2 -T 10 -I -r zap-report.html -J zap-report.json -w zap-report.md -z "$ZAP_OPTS")
echo ">> Ejecutando ZAP (puede tardar varios minutos)"
docker start -a "$CID" | tee reports/zap/zap-console.log
docker cp "$CID":/zap/wrk/. reports/zap/
docker rm -f "$CID" > /dev/null

if [ ! -f reports/zap/zap-report.json ]; then echo "No se genero el reporte"; exit 3; fi
HIGH=$(grep -Eo '"riskcode" *: *"3"' reports/zap/zap-report.json | wc -l)
MEDIUM=$(grep -Eo '"riskcode" *: *"2"' reports/zap/zap-report.json | wc -l)
LOW=$(grep -Eo '"riskcode" *: *"1"' reports/zap/zap-report.json | wc -l)
echo "ZAP -> Alto: $HIGH | Medio: $MEDIUM | Bajo: $LOW" | tee reports/zap/zap-resumen.txt
if [ "$HIGH" -gt 0 ]; then exit 1; fi
exit 0
''')
          if (rc == 1) { unstable('ZAP encontro alertas de riesgo ALTO (esperado en la version vulnerable)') }
          else if (rc != 0) { error("Fallo el escaneo ZAP (codigo ${rc})") }
        }
      }
    }
  }
  post {
    always { archiveArtifacts artifacts: 'reports/**', allowEmptyArchive: true }
  }
}
