pipeline {
    agent any

    parameters {
        file(
            name: 'UPLOADED_BACKUP_FILE',
            description: 'Upload a PostgreSQL backup file to restore'
        )
    }

    environment {
        BACKUP_LOCATION = "${WORKSPACE}/backups/"
        TIMESTAMP = sh(script: 'date +%Y%m%d_%H%M%S', returnStdout: true).trim()
        DB_NAME = "restore_db"       // Change DB name
        DB_HOST = "localhost"         // Change if using remote server
        DB_PORT = "5432"
    }

    stages {

        stage('Initialize') {
            steps {
                script {
                    echo "Starting pipeline..."
                    echo "Workspace: ${WORKSPACE}"

                    sh "mkdir -p ${BACKUP_LOCATION}"

                    if (params.UPLOADED_BACKUP_FILE) {
                        echo "Backup file uploaded. Will restore to PostgreSQL."
                        env.ACTION = 'restore'
                        env.UPLOADED_FILE = "${params.UPLOADED_BACKUP_FILE}"
                    } else {
                        echo "No file uploaded. Will create PostgreSQL backup."
                        env.ACTION = 'backup'
                    }
                }
            }
        }

        /* ========================= RESTORE ========================= */
        stage('Restore PostgreSQL Backup') {
            when { expression { env.ACTION == 'restore' } }
            steps {
                withCredentials([usernamePassword(credentialsId: 'pg-creds', usernameVariable: 'PGUSER', passwordVariable: 'PGPASSWORD')]) {
                    script {
                        echo "Restoring PostgreSQL database from uploaded backup..."

                        sh """
                            export PGHOST=${DB_HOST}
                            export PGPORT=${DB_PORT}
                            export PGDATABASE=${DB_NAME}

                            echo "Restoring file: ${UPLOADED_FILE}"

                            # Detect file type and restore
                            if [[ "${UPLOADED_FILE}" == *.sql ]]; then
                                echo "Using psql to restore .sql"
                                psql < "${UPLOADED_FILE}"

                            elif [[ "${UPLOADED_FILE}" == *.sql.gz ]]; then
                                echo "Using psql to restore .sql.gz"
                                gunzip -c "${UPLOADED_FILE}" | psql

                            elif [[ "${UPLOADED_FILE}" == *.backup ]] || [[ "${UPLOADED_FILE}" == *.dump ]]; then
                                echo "Using pg_restore for custom-format backup"
                                pg_restore -Fc -d "${DB_NAME}" "${UPLOADED_FILE}"

                            else
                                echo "❌ Unsupported backup format!"
                                exit 1
                            fi

                            echo "✓ PostgreSQL Restore Completed Successfully"
                        """

                        currentBuild.description = "Restored PostgreSQL database"
                    }
                }
            }
        }

        /* ========================= BACKUP ========================= */
        stage('Create PostgreSQL Backup') {
            when { expression { env.ACTION == 'backup' } }
            steps {
                withCredentials([usernamePassword(credentialsId: 'pg-creds', usernameVariable: 'PGUSER', passwordVariable: 'PGPASSWORD')]) {
                    script {

                        def backupFileName = "postgres_backup_${TIMESTAMP}.sql.gz"
                        def backupFilePath = "${BACKUP_LOCATION}/${backupFileName}"

                        echo "Creating PostgreSQL backup..."

                        sh """
                            export PGHOST=${DB_HOST}
                            export PGPORT=${DB_PORT}
                            export PGDATABASE=${DB_NAME}

                            echo "Dumping PostgreSQL database: ${DB_NAME}"

                            # Backup in compressed format
                            pg_dump --format=plain | gzip > "${backupFilePath}"

                            if [ -f "${backupFilePath}" ]; then
                                echo "✓ Backup created: ${backupFilePath}"
                                echo "Size: \$(du -h "${backupFilePath}" | cut -f1)"
                            else
                                echo "❌ Backup failed!"
                                exit 1
                            fi
                        """

                        env.BACKUP_CREATED = backupFilePath
                        currentBuild.description = "Backup created: ${backupFileName}"
                    }
                }
            }
        }

        /* ========================= SUMMARY ========================= */
        stage('Summary') {
            steps {
                script {
                    if (env.ACTION == "restore") {
                        echo "=== RESTORE SUMMARY ==="
                        echo "Database restored from file: ${UPLOADED_FILE}"
                    } else {
                        echo "=== BACKUP SUMMARY ==="
                        echo "Backup created at: ${env.BACKUP_CREATED}"
                    }
                }
            }
        }

        /* ========================= ARCHIVE ========================= */
        stage('Archive Backup') {
            when { expression { env.ACTION == 'backup' && env.BACKUP_CREATED } }
            steps {
                archiveArtifacts artifacts: "backups/*.gz", fingerprint: true
            }
        }
    }

    /* ========================= POST ACTIONS ========================= */
    post {
        success {
            echo "Pipeline completed successfully"
            echo "Action: ${env.ACTION}"
        }
        failure {
            echo "Pipeline failed! Check logs."
        }
    }
}
