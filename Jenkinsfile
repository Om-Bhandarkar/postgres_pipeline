pipeline {
    agent any

    parameters {
        // User will upload the backup file when running the job
        file(name: '', description: 'Upload PostgreSQL backup file (.sql or .dump)')
    }

    environment {
        // DB Config
        PG_HOST = "localhost"
        PG_PORT = "5432"
        PG_USER = "postgres"
        PG_DB   = "restore_db"
    }

    stages {
        stage('Copy Backup File to Temp Location') {
            steps {
                script {
                    echo "Copying backup file to workspace..."

                    // Jenkins automatically stores uploaded file into params.DB_BACKUP_FILE
                    backupFilePath = "${workspace}/${params.DB_BACKUP_FILE}"

                    sh "cp '${params.DB_BACKUP_FILE}' '${backupFilePath}'"

                    echo "Backup file saved at: ${backupFilePath}"
                }
            }
        }

        stage('Restore Database') {
            steps {
                script {
                    echo "Restoring PostgreSQL database..."

                    // Detect if file is .sql or .dump
                    if (backupFilePath.endsWith(".sql")) {
                        sh """
                            PGPASSWORD=${PG_PASSWORD} psql \
                            -h ${PG_HOST} -p ${PG_PORT} -U ${PG_USER} -d ${PG_DB} \
                            -f '${backupFilePath}'
                        """
                    } else {
                        // For .dump file
                        sh """
                            PGPASSWORD=${PG_PASSWORD} pg_restore \
                            -h ${PG_HOST} -p ${PG_PORT} -U ${PG_USER} -d ${PG_DB} \
                            '${backupFilePath}'
                        """
                    }

                    echo "Database restore complete."
                }
            }
        }
    }
}

