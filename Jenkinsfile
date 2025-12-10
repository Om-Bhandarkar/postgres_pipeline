pipeline {
  agent any

  parameters {
    file(
      name: 'DB_BACKUP',
      description: 'Upload PostgreSQL Custom Backup File (pg_dump -Fc format)'
    )
    string(
      name: 'DB_NAME',
      defaultValue: 'temp_pg',
      description: 'Database name to restore into'
    )
  }

  environment {
    POSTGRES_PASSWORD = "admin123"
    POSTGRES_USER = "postgres"
  }

  stages {

    stage('Prepare workspace') {
      steps {
        deleteDir()
        sh 'echo "Backup file received:" && ls -l'
      }
    }

    stage('Validate file is PostgreSQL CUSTOM dump') {
      steps {
        script {
          def filePath = "$WORKSPACE/${params.DB_BACKUP}"
          if (!fileExists(filePath)) {
            error "❌ Backup file missing!"
          }

          def type = sh(script: "file -b '${filePath}'", returnStdout: true).trim()
          echo "Detected file type: ${type}"

          if (!type.toLowerCase().contains("postgresql") &&
              !type.toLowerCase().contains("dump")) {
            error "❌ This file is NOT a PostgreSQL custom dump!"
          }

          echo "✔ Custom dump file verified"
        }
      }
    }

    stage('Restore using pg_restore (NO OWNER)') {
      steps {
        script {
          docker.image('postgres:15').inside("--name restore-pg -e POSTGRES_PASSWORD=${POSTGRES_PASSWORD} -e POSTGRES_USER=${POSTGRES_USER} -v $WORKSPACE:/backup") {

            sh '''
              echo "Waiting for PostgreSQL to start…"

              for i in {1..20}; do
                pg_isready -h localhost -U ${POSTGRES_USER} && break
                sleep 2
              done

              FILE="/backup/${DB_BACKUP}"

              echo "Dropping and creating database ${DB_NAME}..."
              psql -U ${POSTGRES_USER} --variable=ON_ERROR_STOP=1 <<EOF
                DROP DATABASE IF EXISTS "${DB_NAME}";
                CREATE DATABASE "${DB_NAME}";
EOF

              echo "Running pg_restore…"

              pg_restore \
                --no-owner \
                --clean \
                --if-exists \
                -U ${POSTGRES_USER} \
                -d ${DB_NAME} \
                "$FILE"

              echo "✔ Restore complete!"
            '''
          }
        }
      }
    }

    stage('Sanity check') {
      steps {
        sh '''
          echo "Tables in ${DB_NAME}:"
          psql -U ${POSTGRES_USER} -d ${DB_NAME} -c "\\dt"
        '''
      }
    }
  }

  post {
    always {
      sh '''
        echo "Cleaning Docker container…"
        docker rm -f restore-pg || true
      '''
    }
  }
}
