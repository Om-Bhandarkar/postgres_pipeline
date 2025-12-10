pipeline {
    agent any

    parameters {
        file(
            name: '',
            description: 'Upload a .tar backup file to restore'
        )
    }

    environment {
        RESTORE_LOCATION = "${WORKSPACE}/restored_files/"
        BACKUP_LOCATION = "${WORKSPACE}/backups/"
        TIMESTAMP = sh(script: 'date +%Y%m%d_%H%M%S', returnStdout: true).trim()
    }

    stages {
        stage('Initialize') {
            steps {
                script {
                    echo "=== Initializing Pipeline ==="
                    echo "Workspace: ${WORKSPACE}"

                    // Create directories
                    sh """
                        mkdir -p "${BACKUP_LOCATION}"
                        mkdir -p "${RESTORE_LOCATION}"
                    """

                    // Detect restore or backup mode
                    if (params.UPLOADED_BACKUP_FILE) {
                        echo "File was uploaded → Restore Mode"
                        env.ACTION = 'restore'
                        env.UPLOADED_FILE = "${params.UPLOADED_BACKUP_FILE}"
                    } else {
                        echo "No file uploaded → Backup Mode"
                        env.ACTION = 'backup'
                    }
                }
            }
        }

        stage('Process Backup / Restore') {
            steps {
                script {
                    if (env.ACTION == 'restore') {

                        echo "=== RESTORE MODE ==="
                        echo "Uploaded file: ${params.UPLOADED_BACKUP_FILE}"

                        // Only allow .tar files
                        sh """
                            echo "Validating uploaded file..."

                            if [ ! -f "${params.UPLOADED_BACKUP_FILE}" ]; then
                                echo "✗ ERROR: File not found!"
                                exit 1
                            fi

                            case "${params.UPLOADED_BACKUP_FILE}" in
                                *.tar)
                                    echo "✓ Valid .tar file detected"
                                    echo "Extracting..."
                                    tar -xf "${params.UPLOADED_BACKUP_FILE}" -C "${RESTORE_LOCATION}"
                                    echo "✓ Restore complete"
                                    ;;
                                *)
                                    echo "✗ ERROR: Only .tar files allowed!"
                                    echo "You uploaded: ${params.UPLOADED_BACKUP_FILE}"
                                    exit 1
                                    ;;
                            esac
                        """

                        currentBuild.description = "Restored backup file"
                    }

                    else {

                        echo "=== BACKUP MODE ==="
                        def backupFileName = "workspace_backup_${TIMESTAMP}.tar.gz"
                        def backupFilePath = "${BACKUP_LOCATION}/${backupFileName}"

                        // Create backup of workspace
                        sh """
                            echo "Creating backup..."
                            tar -czf "${backupFilePath}" .
                            echo "✓ Backup created: ${backupFileName}"
                        """

                        env.BACKUP_CREATED = backupFilePath
                        currentBuild.description = "Backup Created: ${backupFileName}"
                    }
                }
            }
        }

        stage('Summary') {
            steps {
                script {
                    if (env.ACTION == 'restore') {
                        echo "=== RESTORE SUMMARY ==="
                        sh """
                            echo "Restored files:"
                            ls -lh "${RESTORE_LOCATION}"
                        """
                    } else {
                        echo "=== BACKUP SUMMARY ==="
                        echo "Backup file: ${env.BACKUP_CREATED}"
                        sh "ls -lh ${env.BACKUP_CREATED}"
                    }
                }
            }
        }

        stage('Archive Artifacts') {
            steps {
                script {
                    if (env.ACTION == 'backup') {
                        echo "Archiving backup artifact..."
                        archiveArtifacts artifacts: "backups/*.tar.gz", fingerprint: true
                    }
                }
            }
        }
    }

    post {
        success {
            script {
                echo "✔ Pipeline completed successfully!"

                if (env.ACTION == 'backup') {
                    echo "Download backup at:"
                    echo "${BUILD_URL}artifact/backups/"
                } else {
                    echo "Restored files are available at:"
                    echo "${RESTORE_LOCATION}"
                }
            }
        }

        failure {
            echo "✗ Pipeline failed — check console logs!"
        }

        always {
            echo "=== Pipeline End ==="
        }
    }
}
