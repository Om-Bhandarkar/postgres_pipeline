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
    
        /* ----------------- INITIALIZE ----------------- */
        stage('Initialize') {
            steps {
                script {
                    echo "Starting pipeline..."
                    echo "Workspace: ${WORKSPACE}"

                    // Create folders
                    sh """
                        mkdir -p "${RESTORE_LOCATION}"
                        mkdir -p "${BACKUP_LOCATION}"
                    """

                    // IMPORTANT FIX – Jenkins stores uploaded file in WORKSPACE
                    if (params.UPLOADED_BACKUP_FILE) {
                        echo "File was uploaded → Restore mode"
                        env.ACTION = "restore"

                        // FIXED PATH
                        env.UPLOADED_FILE = "${WORKSPACE}/${params.UPLOADED_BACKUP_FILE}"

                        echo "Uploaded file full path: ${env.UPLOADED_FILE}"

                    } else {
                        echo "No file uploaded → Backup mode"
                        env.ACTION = "backup"
                    }

                    env.TIMESTAMP = sh(script: 'date +%Y%m%d_%H%M%S', returnStdout: true).trim()
                }
            }
        }

        /* --------------- PROCESS STAGE ---------------- */
        stage('Process') {
            steps {
                script {

                    /* ============ RESTORE MODE ============ */
                    if (env.ACTION == "restore") {

                        echo "Starting RESTORE..."
                        sh "ls -la ${WORKSPACE}"

                        // Check file actually exists
                        sh """
                            if [ ! -f "${env.UPLOADED_FILE}" ]; then
                                echo "ERROR: Uploaded file not found at ${env.UPLOADED_FILE}"
                                exit 1
                            fi
                        """

                        // Extract or copy file
                        sh """
                            if [[ "${env.UPLOADED_FILE}" == *.tar.gz ]] || [[ "${env.UPLOADED_FILE}" == *.tgz ]]; then
                                echo "Extracting tar.gz..."
                                tar -xzf "${env.UPLOADED_FILE}" -C "${RESTORE_LOCATION}"
                                
                            elif [[ "${env.UPLOADED_FILE}" == *.tar ]]; then
                                echo "Extracting tar..."
                                tar -xf "${env.UPLOADED_FILE}" -C "${RESTORE_LOCATION}"
                                
                            elif [[ "${env.UPLOADED_FILE}" == *.zip ]]; then
                                echo "Extracting zip..."
                                unzip -o "${env.UPLOADED_FILE}" -d "${RESTORE_LOCATION}"
                                
                            else
                                echo "Copying uploaded file as-is..."
                                cp "${env.UPLOADED_FILE}" "${RESTORE_LOCATION}/"
                            fi
                        """

                        echo "✓ Restore completed."
                        currentBuild.description = "RESTORE completed"

                    }
                    
                    /* ============ BACKUP MODE ============ */
                    else {
                        echo "Starting BACKUP..."

                        def backupFileName = "workspace_backup_${env.TIMESTAMP}.tar.gz"
                        def backupFilePath = "${BACKUP_LOCATION}/${backupFileName}"

                        sh """
                            tar -czf "${backupFilePath}" .
                            
                            if [ ! -f "${backupFilePath}" ]; then
                                echo "Backup failed!"
                                exit 1
                            fi

                            echo "✓ Backup created at: ${backupFilePath}"
                        """

                        env.BACKUP_CREATED = backupFilePath
                        currentBuild.description = "BACKUP created"
                    }
                }
            }
        }

        /* ---------------- SUMMARY ---------------- */
        stage('Summary') {
            steps {
                script {
                    if (env.ACTION == "restore") {
                        echo "=== RESTORE SUMMARY ==="
                        sh "ls -la ${RESTORE_LOCATION}"
                    } else {
                        echo "=== BACKUP SUMMARY ==="
                        sh "ls -lh ${env.BACKUP_CREATED}"
                    }
                }
            }
        }

        /* ---------------- ARCHIVE ---------------- */
        stage('Archive Backup') {
            when { expression { env.ACTION == 'backup' } }
            steps {
                archiveArtifacts artifacts: "backups/*", fingerprint: true
            }
        }
    }
    
    post {
        success {
            echo "Pipeline completed successfully."
        }
        failure {
            echo "Pipeline FAILED. Check above logs."
        }
    }
}
