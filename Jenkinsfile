pipeline {
    agent any
    
    parameters {
        file(
            name: 'UPLOADED_BACKUP_FILE',
            description: 'Upload a backup file to restore'
        )
    }
    
    environment {
        RESTORE_LOCATION = '/opt/restored_files/'
        BACKUP_LOCATION = '/opt/backups/'
        TIMESTAMP = sh(script: 'date +%Y%m%d_%H%M%S', returnStdout: true).trim()
    }
    
    stages {
        stage('Initialize') {
            steps {
                script {
                    echo "Starting pipeline..."
                    
                    // Create directories if they don't exist
                    sh """
                        mkdir -p "${BACKUP_LOCATION}"
                        mkdir -p "${RESTORE_LOCATION}"
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
                        
                        // The uploaded file is in the workspace
                        def uploadedFilePath = "${env.UPLOADED_FILE}"
                        
                        sh """
                            echo "Uploaded file path: ${uploadedFilePath}"
                            echo "File exists check:"
                            ls -la "${uploadedFilePath}" || true
                        """
                        
                        // Move uploaded file to restore location and extract
                        sh """
                            # Create restore directory
                            mkdir -p "${RESTORE_LOCATION}"
                            
                            # Check if file is a tar.gz archive
                            if file "${uploadedFilePath}" | grep -q "gzip compressed data"; then
                                echo "Extracting gzip compressed backup..."
                                tar -xzf "${uploadedFilePath}" -C "${RESTORE_LOCATION}"
                            elif file "${uploadedFilePath}" | grep -q "tar archive"; then
                                echo "Extracting tar archive..."
                                tar -xf "${uploadedFilePath}" -C "${RESTORE_LOCATION}"
                            elif file "${uploadedFilePath}" | grep -q "Zip archive"; then
                                echo "Extracting zip archive..."
                                unzip "${uploadedFilePath}" -d "${RESTORE_LOCATION}"
                            else
                                echo "Copying as regular file..."
                                cp "${uploadedFilePath}" "${RESTORE_LOCATION}/"
                            fi
                            
                            echo "✓ Restoration completed successfully"
                            echo "Restored files in ${RESTORE_LOCATION}:"
                            ls -la "${RESTORE_LOCATION}/"
                        """
                        
                        currentBuild.description = "Restored uploaded backup"
                        
                    } else {
                        // BACKUP: Create backup from workspace
                        echo "Creating backup from workspace contents..."
                        
                        def backupFileName = "workspace_backup_${TIMESTAMP}.tar.gz"
                        def backupFilePath = "${BACKUP_LOCATION}/${backupFileName}"
                        
                        sh """
                            # Create backup of current workspace
                            tar -czf "${backupFilePath}" .
                            
                            # Verify backup
                            if [ -f "${backupFilePath}" ]; then
                                echo "✓ Backup created successfully"
                                echo "Backup location: ${backupFilePath}"
                                echo "Backup size: \$(du -h "${backupFilePath}" | cut -f1)"
                                echo "Backup MD5: \$(md5sum "${backupFilePath}" | cut -d' ' -f1)"
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
                            find "${RESTORE_LOCATION}" -type f | head -20
                            echo ""
                            echo "Directory structure:"
                            tree "${RESTORE_LOCATION}" -L 3 2>/dev/null || find "${RESTORE_LOCATION}" -type d | head -20
                        """
                    } else {
                        echo "=== BACKUP SUMMARY ==="
                        echo "Action: Create backup from workspace"
                        echo "Backup file: ${env.BACKUP_CREATED}"
                        sh """
                            echo "Backup details:"
                            ls -lh "${env.BACKUP_CREATED}"
                            echo ""
                            echo "All backups in ${BACKUP_LOCATION}:"
                            ls -lh "${BACKUP_LOCATION}" | head -10
                        """
                    }
                }
            }
        }
        
        stage('Optional: Download Backup') {
            when {
                expression { env.ACTION == 'backup' && env.BACKUP_CREATED }
            }
            steps {
                script {
                    echo "Backup created and available for download:"
                    echo "File: ${env.BACKUP_CREATED}"
                    
                    // Archive the backup file as an artifact
                    archiveArtifacts artifacts: "${env.BACKUP_CREATED}", fingerprint: true
                    
                    sh """
                        echo "Backup file size: \$(du -h "${env.BACKUP_CREATED}" | cut -f1)"
                        echo "MD5 checksum: \$(md5sum "${env.BACKUP_CREATED}" | cut -d' ' -f1)"
                    """
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
                    echo "You can download it from the Jenkins artifacts"
                } else {
                    echo "File restored to: ${RESTORE_LOCATION}"
                }
            }
        }
        
        failure {
            echo "Pipeline failed! Check console output for details."
        }
        
        always {
            echo "Cleaning up..."
            // Optional: clean workspace but keep important files
            cleanWs(cleanWhenAborted: true, cleanWhenFailure: true, cleanWhenNotBuilt: true, cleanWhenUnstable: true, cleanWhenSuccess: true)
        }
    }
}
