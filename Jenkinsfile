pipeline {
    agent any
    
    parameters {
        file(
            name: 'UPLOADED_BACKUP_FILE',
            description: 'Upload a backup file to restore'
        )
    }
    
    environment {
        // Use workspace directories instead of /opt/
        RESTORE_LOCATION = "${WORKSPACE}/restored_files/"
        BACKUP_LOCATION = "${WORKSPACE}/backups/"
        TIMESTAMP = sh(script: 'date +%Y%m%d_%H%M%S', returnStdout: true).trim()
    }
    
    stages {
        stage('Initialize') {
            steps {
                script {
                    echo "Starting pipeline..."
                    echo "Workspace: ${WORKSPACE}"
                    
                    // Create directories in workspace (no permission issues)
                    sh """
                        mkdir -p "${BACKUP_LOCATION}"
                        mkdir -p "${RESTORE_LOCATION}"
                        echo "Backup location: ${BACKUP_LOCATION}"
                        echo "Restore location: ${RESTORE_LOCATION}"
                    """
                    
                    // Check if a file was uploaded
                    if (params.UPLOADED_BACKUP_FILE) {
                        echo "File was uploaded by user"
                        env.ACTION = 'restore'
                        env.UPLOADED_FILE = "${params.UPLOADED_BACKUP_FILE}"
                    } else {
                        echo "No file uploaded. Creating backup from workspace..."
                        env.ACTION = 'backup'
                    }
                }
            }
        }
        
        stage('Process Backup/Restore') {
            steps {
                script {
                    if (env.ACTION == 'restore') {
                        // RESTORE: Process uploaded file
                        echo "Processing uploaded backup file for restoration..."
                        
                        sh """
                            echo "Uploaded file: ${params.UPLOADED_BACKUP_FILE}"
                            echo "Listing workspace files:"
                            pwd
                            ls -la
                        """
                        
                        // Extract uploaded file
                        sh """
                            # Create restore directory
                            mkdir -p "${RESTORE_LOCATION}"
                            
                            # Check file type and extract
                            if [ -f "${params.UPLOADED_BACKUP_FILE}" ]; then
                                echo "File found, extracting..."
                                
                                if [[ "${params.UPLOADED_BACKUP_FILE}" == *.tar.gz ]] || [[ "${params.UPLOADED_BACKUP_FILE}" == *.tgz ]]; then
                                    echo "Extracting tar.gz file..."
                                    tar -xzf "${params.UPLOADED_BACKUP_FILE}" -C "${RESTORE_LOCATION}"
                                elif [[ "${params.UPLOADED_BACKUP_FILE}" == *.tar ]]; then
                                    echo "Extracting tar file..."
                                    tar -xf "${params.UPLOADED_BACKUP_FILE}" -C "${RESTORE_LOCATION}"
                                elif [[ "${params.UPLOADED_BACKUP_FILE}" == *.zip ]]; then
                                    echo "Extracting zip file..."
                                    unzip "${params.UPLOADED_BACKUP_FILE}" -d "${RESTORE_LOCATION}"
                                else
                                    echo "Copying as regular file..."
                                    cp "${params.UPLOADED_BACKUP_FILE}" "${RESTORE_LOCATION}/"
                                fi
                                
                                echo "✓ Restoration completed"
                            else
                                echo "Uploaded file not found!"
                                exit 1
                            fi
                        """
                        
                        currentBuild.description = "Restored uploaded backup"
                        
                    } else {
                        // BACKUP: Create backup from workspace
                        echo "Creating backup from workspace contents..."
                        
                        def backupFileName = "workspace_backup_${TIMESTAMP}.tar.gz"
                        def backupFilePath = "${BACKUP_LOCATION}/${backupFileName}"
                        
                        sh """
                            echo "Creating backup of workspace..."
                            echo "Current directory:"
                            pwd
                            echo "Workspace contents:"
                            ls -la
                            
                            # Create backup of current workspace
                            tar -czf "${backupFilePath}" .
                            
                            # Verify backup
                            if [ -f "${backupFilePath}" ]; then
                                echo "✓ Backup created successfully"
                                echo "Backup location: ${backupFilePath}"
                                echo "Backup size: \$(du -h "${backupFilePath}" | cut -f1)"
                            else
                                echo "✗ Backup failed!"
                                exit 1
                            fi
                        """
                        
                        currentBuild.description = "Backup created: ${backupFileName}"
                        env.BACKUP_CREATED = backupFilePath
                    }
                }
            }
        }
        
        stage('Summary') {
            steps {
                script {
                    if (env.ACTION == 'restore') {
                        echo "=== RESTORE SUMMARY ==="
                        echo "Action: Restore uploaded file"
                        echo "Restore location: ${RESTORE_LOCATION}"
                        sh """
                            echo "Restored contents:"
                            ls -la "${RESTORE_LOCATION}/"
                            echo ""
                            echo "Total files: \$(find "${RESTORE_LOCATION}" -type f | wc -l)"
                        """
                    } else {
                        echo "=== BACKUP SUMMARY ==="
                        echo "Action: Create backup from workspace"
                        echo "Backup file: ${env.BACKUP_CREATED}"
                        sh """
                            echo "Backup details:"
                            ls -lh "${env.BACKUP_CREATED}"
                        """
                    }
                }
            }
        }
        
        stage('Archive Artifacts') {
            steps {
                script {
                    if (env.ACTION == 'backup' && env.BACKUP_CREATED) {
                        echo "Archiving backup as Jenkins artifact..."
                        archiveArtifacts artifacts: "backups/*", fingerprint: true
                    }
                }
            }
        }
    }
    
    post {
        success {
            script {
                echo "Pipeline completed successfully!"
                echo "Action: ${env.ACTION}"
                
                if (env.ACTION == 'backup') {
                    echo "Backup created: ${env.BACKUP_CREATED}"
                    echo "Download from: ${BUILD_URL}artifact/backups/"
                } else {
                    echo "Files restored to workspace directory: ${RESTORE_LOCATION}"
                }
            }
        }
        
        failure {
            echo "Pipeline failed! Check console output for details."
        }
        
        always {
            echo "Pipeline execution completed."
        }
    }
}
