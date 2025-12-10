pipeline {
    agent any

    parameters {
        file(
            name: 'UPLOADED_BACKUP_FILE',
            description: 'Upload a backup file to restore'
        )
    }

    environment {
        RESTORE_LOCATION = "${WORKSPACE}/restored_files"
        BACKUP_LOCATION  = "${WORKSPACE}/backups"
    }

    stages {

        /* ------------------ INITIALIZE ------------------ */
        stage('Initialize') {
            steps {
                script {
                    echo "Initializing pipeline..."
                    echo "Workspace: ${WORKSPACE}"

                    sh """
                        mkdir -p "${BACKUP_LOCATION}"
                        mkdir -p "${RESTORE_LOCATION}"
                    """

                    if (params.UPLOADED_BACKUP_FILE) {
                        echo "Backup file uploaded → Restore mode"
                        env.ACTION = "restore"
                        env.UPLOADED_FILE = params.UPLOADED_BACKUP_FILE
                    } else {
                        echo "No file uploaded → Backup mode"
                        env.ACTION = "backup"
                    }

                    env.TIMESTAMP = sh(
                        script: 'date +%Y%m%d_%H%M%S',
                        returnStdout: true
                    ).trim()
                }
            }
        }

        /* ------------------ PROCESS ------------------ */
        stage('Process Backup/Restore') {
            steps {
                script {

                    /* ===== RESTORE MODE ===== */
                    if (env.ACTION == "restore") {
                        echo "Starting restore process..."

                        sh """
                            echo "Uploaded backup file: ${env.UPLOADED_FILE}"
                            ls -la

                            mkdir -p "${RESTORE_LOCATION}"

                            if [ ! -f "${env.UPLOADED_FILE}" ]; then
                                echo "ERROR: Uploaded file not found."
                                exit 1
                            fi

                            case "${env.UPLOADED_FILE}" in
                                *.tar.gz|*.tgz)
                                    echo "Extracting tar.gz..."
                                    tar -xzf "${env.UPLOADED_FILE}" -C "${RESTORE_LOCATION}"
                                    ;;
                                *.tar)
                                    echo "Extracting tar..."
                                    tar -xf "${env.UPLOADED_FILE}" -C "${RESTORE_LOCATION}"
                                    ;;
                                *.zip)
                                    echo "Extracting zip..."
                                    unzip -o "${env.UPLOADED_FILE}" -d "${RESTORE_LOCATION}"
                                    ;;
                                *)
                                    echo "Regular file detected → Copying"
                                    cp "${env.UPLOADED_FILE}" "${RESTORE_LOCATION}/"
                                    ;;
                            esac

                            echo "✓ Restore completed"
                        """

                        currentBuild.description = "Restored uploaded backup"

                    } else {

                        /* ===== BACKUP MODE ===== */
                        echo "Starting workspace backup..."

                        def backupFileName = "workspace_backup_${env.TIMESTAMP}.tar.gz"
                        def backupFilePath = "${BACKUP_LOCATION}/${backupFileName}"

                        sh """
                            echo "Backing up workspace..."
                            tar -czf "${backupFilePath}" .

                            if [ ! -f "${backupFilePath}" ]; then
                                echo "Backup failed!"
                                exit 1
                            fi

                            echo "✓ Backup created: ${backupFilePath}"
                            du -h "${backupFilePath}"
                        """

                        env.BACKUP_CREATED = backupFilePath
                        currentBuild.description = "Backup created: ${backupFileName}"
                    }
                }
            }
        }

        /* ------------------ SUMMARY ------------------ */
        stage('Summary') {
            steps {
                script {

                    if (env.ACTION == "restore") {
                        echo "===== RESTORE SUMMARY ====="
                        echo "Restored to: ${RESTORE_LOCATION}"

                        sh """
                            echo "Restored files:"
                            ls -la "${RESTORE_LOCATION}"
                            echo "File count: \$(find "${RESTORE_LOCATION}" -type f | wc -l)"
                        """

                    } else {
                        echo "===== BACKUP SUMMARY ====="
                        echo "Backup file: ${env.BACKUP_CREATED}"

                        sh """
                            ls -lh "${env.BACKUP_CREATED}"
                        """
                    }
                }
            }
        }

        /* ------------------ ARCHIVE ARTIFACTS ------------------ */
        stage('Archive Backup') {
            when { expression { env.ACTION == 'backup' } }
            steps {
                echo "Archiving backup artifacts..."
                archiveArtifacts artifacts: 'backups/*', fingerprint: true
            }
        }
    }

    /* ------------------ POST STATUS ------------------ */
    post {
        success {
            script {
                echo "Pipeline completed successfully!"
                echo "Action: ${env.ACTION}"

                if (env.ACTION == "backup") {
                    echo "Backup saved at: ${env.BACKUP_CREATED}"
                } else {
                    echo "Restore completed. Files in: ${RESTORE_LOCATION}"
                }
            }
        }

        failure {
            echo "Pipeline failed. Check logs."
        }

        always {
            echo "Pipeline finished execution."
        }
    }
}
