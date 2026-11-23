pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
    }

    triggers {
        pollSCM('H/2 * * * *')
    }

    environment {
        APP_PORT = '5001'
        POSTGRES_PORT = '5433'
        POSTGRES_DB = 'tareas'
        POSTGRES_USER = 'tareas_user'
        POSTGRES_PASSWORD = credentials('postgres-ci-password')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'docker compose build'
            }
        }

        stage('Test') {
            steps {
                sh 'docker compose up -d --wait'
                sh '''
                    docker compose exec -T app python -c '
                    import json, urllib.request
                    payload = json.dumps({"titulo": "Prueba Jenkins"}).encode()
                    request = urllib.request.Request(
                        "http://localhost:5000/tareas",
                        data=payload,
                        headers={"Content-Type": "application/json"},
                        method="POST",
                    )
                    created = json.load(urllib.request.urlopen(request))
                    assert created["titulo"] == "Prueba Jenkins"
                    tasks = json.load(urllib.request.urlopen("http://localhost:5000/tareas"))
                    assert any(task["titulo"] == "Prueba Jenkins" for task in tasks)
                    print("Integración app -> PostgreSQL: OK")
                    '
                '''
            }
        }
    }

    post {
        always {
            sh 'docker compose down -v --remove-orphans || true'
        }
        success {
            echo 'Pipeline completado correctamente.'
        }
        failure {
            echo 'Pipeline fallido. Revisar la salida de consola.'
        }
    }
}
