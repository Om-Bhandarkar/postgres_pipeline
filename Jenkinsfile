pipeline {
    agent any

    parameters {
        file(description: 'Upload PostgreSQL backup file')
    }

    environment {
        PG_HOST = 'localhost'
        PG_PORT = '5433'
        PG_DB   = 'restore_db'
    }

    stages {

        stage('Receive Backup File') {
            steps {
                script {
                    echo "Receiving uploaded backup file..."

                    // Uploaded file is placed automatically in workspace
                    sh """
                        if [ ! -f "${params.BACKUP_FILE}" ]; then
                            echo '❌ ERROR: Backup file not uploaded or not found.'
                            exit 1
                        fi
                    """

                    echo "✔ Backup file received: ${params.BACKUP_FILE}"
                }
            }
        }

        stage('Restore PostgreSQL') {
            environment {
                DB_CREDS = credentials('a5bde45d-3b6d-495d-a022-14f7f3f977ba')
            }
            steps {
                script {
                    echo "Restoring database from uploaded file..."

                    sh """
                        export PGPASSWORD="${DB_CREDS_PSW}"
                        psql -h ${PG_HOST} -p ${PG_PORT} -U ${DB_CREDS_USR} -d ${PG_DB} -f "${params.BACKUP_FILE}"
                    """

                    echo "✔ Database Restore Complete"
                }
            }
        }

        stage('Verify Restore') {
            environment {
                DB_CREDS = credentials('a5bde45d-3b6d-495d-a022-14f7f3f977ba')
            }
            steps {
                script {
                    echo "Verifying restore..."

                    sh """
                        export PGPASSWORD="${DB_CREDS_PSW}"
                        psql -h ${PG_HOST} -p ${PG_PORT} -U ${DB_CREDS_USR} -d ${PG_DB} -c "\\dt"
                    """
                }
            }
        }
    }

    post {
        success { echo "🎉 Pipeline completed successfully!" }
        failure { echo "❌ Pipeline failed. Check logs!" }
    }
}
