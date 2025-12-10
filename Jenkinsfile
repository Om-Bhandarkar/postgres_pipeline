pipeline {
    agent any

    parameters {
        file(
            
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

                    // Check if uploaded file actually exists
                    def uploaded = params.UPLOADED_BACKUP_FILE
                    def exists = sh(script: "test -f \"${uploaded}\" && echo yes || echo no", returnStdout: true).trim()

                    if (exists == "yes") {
                        echo "Backup file uploaded → RESTORE MODE"
                        env.ACTION = 'restore'

                        // Copy uploaded file to a predictable path
                        sh """
                            cp "${uploaded}" "${WORKSPACE}/uploaded_backup_file"
                        """

                        env.UPLOADED_FILE = "${WORKSPACE}/uploaded_backup_file"

                    } else {
                        echo "No file uploaded → BACKUP MODE"
                        env.ACTION = 'backup'

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

                        sh """
                            set -e
                            export PGHOST=${DB_HOST}
                            export PGPORT=${DB_PORT}
                            export PGDATABASE=${DB_NAME}

                            FILE="${env.UPLOADED_FILE}"

                            echo "Using uploaded backup: \$FILE"

                            case "\$FILE" in
                                *.sql)
                                    psql < "\$FILE"
                                    ;;
                                *.sql.gz)
                                    gunzip -c "\$FILE" | psql
                                    ;;
                                *.backup|*.dump)
                                    pg_restore -Fc -d "${DB_NAME}" "\$FILE"
                                    ;;
                                *)
                                    echo "❌ Unsupported file format!"
                                    exit 1
                                    ;;
                            esac

                            echo "✔ Restore Completed"
                        """

                        currentBuild.description = "RESTORED using uploaded file"
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
                        """

                        env.BACKUP_CREATED = backupFilePath
                        currentBuild.description = "BACKUP: ${backupFileName}"
                    }
                }
            }
        }

        /* ===================== SUMMARY ===================== */
        stage('Summary') {
            steps {
                script {
                    if (env.ACTION == 'restore') {
                        echo "RESTORED FILE : ${env.UPLOADED_FILE}"
                    } else {
                        echo "BACKUP CREATED : ${env.BACKUP_CREATED}"
                    }
                }
            }
        }

        /* ===================== ARCHIVE BACKUP ===================== */
        stage('Archive Backup') {
            when { expression { env.ACTION == 'backup' && env.BACKUP_CREATED } }

            steps {
                archiveArtifacts artifacts: "backups/*.gz", fingerprint: true
            }
        }
    }

    post {
        success { echo "Pipeline completed successfully ✔" }
        failure { echo "Pipeline failed ❌" }
    }
}
