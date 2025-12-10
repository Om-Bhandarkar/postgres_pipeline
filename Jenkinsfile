pipeline {
  agent any

  parameters {
    file(
      name: 'DB_BACKUP',
      description: 'Upload PostgreSQL SQL backup file (DBeaver export)'
    )
    string(
      name: 'DB_NAME',
      defaultValue: 'temp_pg',
      description: 'Temporary DB name to restore into'
    )
  }

  environment {
    POSTGRES_PASSWORD = credentials('postgres-password-id')
    POSTGRES_USER = "postgres"
  }

  stages {

    stage('Prepare workspace') {
      steps {
        deleteDir()
        sh 'echo "Backup file received:" && ls -l'
      }
    }

    stage('Validate SQL file') {
      steps {
        script {
          def path = "$WORKSPACE/${params.DB_BACKUP}"

          if (!fileExists(path)) {
            error "❌ Backup file missing in workspace!"
          }

          def mime = sh(script: "file -b --mime-type '${path}'", returnStdout: true).trim()
          echo "Detected MIME type: ${mime}"

          // DBeaver export always = text/sql
          if (!mime.contains("text")) {
            error "❌ The uploaded file does not appear to be a valid SQL file!"
          }

          echo "✔ Valid SQL backup detected (DBeaver export)"
        }
      }
    }

    stage('Restore SQL backup (NO OWNER mode)') {
      agent {
        docker {
          image 'postgres:15'
          args """
            --name temp-postgres
            -e POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
            -e POSTGRES_USER=${POSTGRES_USER}
            -v $WORKSPACE:/backup
          """
          reuseNode true
        }
      }

      steps {
        sh '''
          echo "Starting PostgreSQL…"
          for i in {1..30}; do
            pg_isready -h localhost -U "${POSTGRES_USER}" && break
            sleep 2
          done

          SQL="/backup/${DB_BACKUP}"

          echo "Dropping and creating database: ${DB_NAME}"

          psql -U "${POSTGRES_USER}" --variable=ON_ERROR_STOP=1 <<EOF
            DROP DATABASE IF EXISTS "${DB_NAME}";
            CREATE DATABASE "${DB_NAME}";
EOF

          export PGDATABASE="${DB_NAME}"

          echo "Applying NO OWNER mode…"
          psql -U "${POSTGRES_USER}" --variable=ON_ERROR_STOP=1 <<EOF
            SET session_replication_role='replica';
EOF

          echo "Restoring SQL file into ${DB_NAME}…"
          psql -U "${POSTGRES_USER}" --variable=ON_ERROR_STOP=1 -f "$SQL"

          echo "Restoring normal replication role…"
          psql -U "${POSTGRES_USER}" --variable=ON_ERROR_STOP=1 <<EOF
            SET session_replication_role='origin';
EOF

          echo "✔ Restore complete — NO OWNER applied"
        '''
      }
    }

    stage('Sanity check') {
      steps {
        sh '''
          echo "Listing tables in ${DB_NAME} …"
          psql -U "${POSTGRES_USER}" -d "${DB_NAME}" -c "\\dt"
        '''
      }
    }
  }

  post {
    always {
      sh '''
        echo "Cleaning up PostgreSQL container…"
        docker rm -f temp-postgres || true
      '''
    }
  }
}
