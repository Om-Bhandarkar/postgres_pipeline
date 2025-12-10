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
        DB_NAME = "postgres"        // Change DB name if needed
        DB_HOST = "localhost"
        DB_PORT = "5432"
        PG_CREDS = "0a5c8c57-970e-40bf-8058-37555d6a2a52"
    }

    stages {

        /* ===================== INIT ===================== */
        stage('Initialize') {
            steps {
                script {
                    echo "Workspace: ${WORKSPACE}"

                    sh "mkdir -p ${BACKUP_LOCATION}"

                    if (params.UPLOADED_BACKUP_FILE) {
                        echo "Backup file uploaded → RESTORE MODE"
                        env.ACTION = 'restore'
                        env.UPLOADED_FILE = "${params.UPLOADED_BACKUP_FILE}"
                    } else {
                        echo "No file uploaded → BACKUP MODE"
                        env.ACTION = 'backup'
                    }
                }
            }
        }

        /* ===================== RESTORE ===================== */
        stage('Restore PostgreSQL Backup') {
            when { expression { env.ACTION == 'restore' } }

            steps {
                withCredentials([usernamePassword(credentialsId: "${PG_CREDS}", usernameVariable: 'PGUSER', passwordVariable: 'PGPASSWORD')]) {
                    script {
                        echo "Restoring PostgreSQL database using uploaded file..."

                        sh """
                            export PGHOST=${DB_HOST}
                            export PGPORT=${DB_PORT}
                            export PGDATABASE=${DB_NAME}

                            echo "Uploaded: ${UPLOADED_FILE}"

                            if [[ "${UPLOADED_FILE}" == *.sql ]]; then
                                echo "Detected .sql → Restoring with psql"
                                psql < "${UPLOADED_FILE}"

                            elif [[ "${UPLOADED_FILE}" == *.sql.gz ]]; then
                                echo "Detected .sql.gz → gunzip + psql"
                                gunzip -c "${UPLOADED_FILE}" | psql

                            elif [[ "${UPLOADED_FILE}" == *.backup || "${UPLOADED_FILE}" == *.dump ]]; then
                                echo "Detected custom-format → pg_restore"
                                pg_restore -Fc -d "${DB_NAME}" "${UPLOADED_FILE}"

                            else
                                echo "❌ Unsupported backup file format!"
                                exit 1
                            fi

                            echo "✔ PostgreSQL Restore Completed"
                        """

                        currentBuild.description = "Restored PostgreSQL DB"
                    }
                }
            }
        }

        /* ===================== BACKUP ===================== */
        stage('Create PostgreSQL Backup') {
            when { expression { env.ACTION == 'backup' } }

            steps {
                withCredentials([usernamePassword(credentialsId: "${PG_CREDS}", usernameVariable: 'PGUSER', passwordVariable: 'PGPASSWORD')]) {
                    script {

                        def backupFileName = "postgres_backup_${TIMESTAMP}.sql.gz"
                        def backupFilePath = "${BACKUP_LOCATION}/${backupFileName}"

                        echo "Creating backup: ${backupFileName}"

                        sh """
                            export PGHOST=${DB_HOST}
                            export PGPORT=${DB_PORT}
                            export PGDATABASE=${DB_NAME}

                            echo "Running pg_dump..."
                            pg_dump --format=plain | gzip > "${backupFilePath}"

                            if [ -f "${backupFilePath}" ]; then
                                echo "✔ Backup created successfully"
                                echo "Location: ${backupFilePath}"
                            else
                                echo "❌ Backup failed"
                                exit 1
                            fi
                        """

                        env.BACKUP_CREATED = backupFilePath
                        currentBuild.description = "Backup: ${backupFileName}"
                    }
                }
            }
        }

        /* ===================== SUMMARY ===================== */
        stage('Summary') {
            steps {
                script {
                    if (env.ACTION == "restore") {
                        echo "=== RESTORE SUMMARY ==="
                        echo "Restored file: ${env.UPLOADED_FILE}"
                    } else {
                        echo "=== BACKUP SUMMARY ==="
                        echo "Backup created: ${env.BACKUP_CREATED}"
                    }
                }
            }
        }

        /* ===================== ARCHIVE ===================== */
        stage('Archive Backup') {
            when { expression { env.ACTION == 'backup' && env.BACKUP_CREATED } }

            steps {
                archiveArtifacts artifacts: "backups/*.gz", fingerprint: true
            }
        }
    }

    /* ===================== POST ACTIONS ===================== */
    post {
        success {
            echo "Pipeline Completed Successfully ✔"
        }
        failure {
            echo "Pipeline Failed ❌"
        }
    }
}
