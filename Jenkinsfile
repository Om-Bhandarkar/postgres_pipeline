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
        DB_NAME = "postgres"
        DB_HOST = "localhost"
        DB_PORT = "5432"
        PG_CREDS = "0a5c8c57-970e-40bf-8058-37555d6a2a52"
    }

    stages {

        /* ===================== INIT ===================== */
        stage('Initialize') {
            steps {
                script {
                    sh "mkdir -p ${BACKUP_LOCATION}"

                    if (params.UPLOADED_BACKUP_FILE) {
                        echo "Backup file uploaded → RESTORE MODE"
                        env.ACTION = 'restore'

                        // COPY uploaded file to a known, safe path
                        sh """
                            cp "${params.UPLOADED_BACKUP_FILE}" "${WORKSPACE}/uploaded_backup_file"
                        """

                        env.UPLOADED_FILE = "${WORKSPACE}/uploaded_backup_file"

                    } else {
                        echo "No file uploaded → BACKUP MODE"
                        env.ACTION = 'backup'

                        // Generate timestamp dynamically inside stage (environment block cannot use sh)
                        env.TIMESTAMP = sh(script: 'date +%Y%m%d_%H%M%S', returnStdout: true).trim()
                    }
                }
            }
        }

        /* ===================== RESTORE ===================== */
        stage('Restore PostgreSQL Backup') {
            when { expression { env.ACTION == 'restore' } }

            steps {
                withCredentials([usernamePassword(credentialsId: PG_CREDS, usernameVariable: 'PGUSER', passwordVariable: 'PGPASSWORD')]) {
                    script {
                        echo "Restoring PostgreSQL database..."

                        sh """
                            set -e
                            export PGHOST=${DB_HOST}
                            export PGPORT=${DB_PORT}
                            export PGDATABASE=${DB_NAME}

                            FILE="${env.UPLOADED_FILE}"

                            echo "Using uploaded file: \$FILE"

                            case "\$FILE" in
                                *.sql)
                                    echo "Detected SQL → using psql"
                                    psql < "\$FILE"
                                    ;;
                                *.sql.gz)
                                    echo "Detected SQL.GZ → gunzip + psql"
                                    gunzip -c "\$FILE" | psql
                                    ;;
                                *.backup|*.dump)
                                    echo "Detected custom format → pg_restore"
                                    pg_restore -Fc -d "${DB_NAME}" "\$FILE"
                                    ;;
                                *)
                                    echo "❌ Unsupported backup file format!"
                                    exit 1
                                    ;;
                            esac

                            echo "✔ PostgreSQL Restore Completed"
                        """

                        currentBuild.description = "Restored PostgreSQL from uploaded file"
                    }
                }
            }
        }

        /* ===================== BACKUP ===================== */
        stage('Create PostgreSQL Backup') {
            when { expression { env.ACTION == 'backup' } }

            steps {
                withCredentials([usernamePassword(credentialsId: PG_CREDS, usernameVariable: 'PGUSER', passwordVariable: 'PGPASSWORD')]) {
                    script {

                        def backupFileName = "postgres_backup_${env.TIMESTAMP}.sql.gz"
                        def backupFilePath = "${BACKUP_LOCATION}/${backupFileName}"

                        sh """
                            export PGHOST=${DB_HOST}
                            export PGPORT=${DB_PORT}
                            export PGDATABASE=${DB_NAME}

                            echo "Creating backup: ${backupFileName}"

                            pg_dump --format=plain | gzip > "${backupFilePath}"

                            if [ -f "${backupFilePath}" ]; then
                                echo "✔ Backup created successfully"
                            else
                                echo "❌ Backup creation failed"
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

    post {
        success {
            echo "Pipeline Completed Successfully ✔"
        }
        failure {
            echo "Pipeline Failed ❌"
        }
    }
}
