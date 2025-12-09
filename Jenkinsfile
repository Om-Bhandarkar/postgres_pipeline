pipeline {
    agent any
    
    parameters {
        string(
            name: 'BACKUP_FILE',
            defaultValue: '',
            description: 'Full path to the backup file (e.g., /opt/backups/mybackup.tar.gz)'
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
                    echo "Backup File: ${params.BACKUP_FILE}"
                    
                    // Create directories if they don't exist
                    sh """
                        mkdir -p "${BACKUP_LOCATION}"
                        mkdir -p "${RESTORE_LOCATION}"
                    """
                }
            }
        }
        
        stage('Validate Backup File') {
            steps {
                script {
                    if (!params.BACKUP_FILE?.trim()) {
                        error "BACKUP_FILE parameter is required"
                    }
                    
                    // Check if backup file exists
                    if (!fileExists(params.BACKUP_FILE)) {
                        echo "Backup file not found: ${params.BACKUP_FILE}"
                        echo "Creating a new backup..."
                        
                        // Extract filename from path
                        def fileName = new File(params.BACKUP_FILE).getName()
                        def backupFilePath = "${BACKUP_LOCATION}/${fileName}"
                        
                        echo "Will create backup at: ${backupFilePath}"
                        env.BACKUP_PATH = backupFilePath
                        env.ACTION = 'backup'
                    } else {
                        echo "Backup file exists: ${params.BACKUP_FILE}"
                        echo "Will restore from this backup..."
                        env.BACKUP_PATH = params.BACKUP_FILE
                        env.ACTION = 'restore'
                    }
                }
            }
        }
        
        stage('Backup or Restore') {
            steps {
                script {
                    if (env.ACTION == 'backup') {
                        stage('Create New Backup') {
                            steps {
                                script {
                                    echo "Creating new backup at: ${env.BACKUP_PATH}"
                                    
                                    // Check if we're backing up a file or directory
                                    def sourcePath = params.BACKUP_FILE
                                    if (params.BACKUP_FILE.contains('/')) {
                                        sourcePath = new File(params.BACKUP_FILE).getParent()
                                    }
                                    
                                    // Create backup with tar.gz compression
                                    sh """
                                        # Create backup
                                        tar -czf "${env.BACKUP_PATH}" -C "${sourcePath}" .
                                        
                                        # Verify the backup was created
                                        if [ -f "${env.BACKUP_PATH}" ]; then
                                            echo "✓ Backup created successfully"
                                            echo "Backup location: ${env.BACKUP_PATH}"
                                            echo "Backup size: \$(du -h "${env.BACKUP_PATH}" | cut -f1)"
                                            echo "Backup MD5: \$(md5sum "${env.BACKUP_PATH}" | cut -d' ' -f1)"
                                        else
                                            echo "✗ Backup failed!"
                                            exit 1
                                        fi
                                    """
                                    
                                    currentBuild.description = "Backup created: ${env.BACKUP_PATH}"
                                }
                            }
                        }
                    } else {
                        stage('Restore from Backup') {
                            steps {
                                script {
                                    echo "Restoring from backup: ${env.BACKUP_PATH}"
                                    echo "Restoring to: ${RESTORE_LOCATION}"
                                    
                                    // Extract the backup
                                    sh """
                                        # Create restore directory if it doesn't exist
                                        mkdir -p "${RESTORE_LOCATION}"
                                        
                                        # Extract the backup
                                        tar -xzf "${env.BACKUP_PATH}" -C "${RESTORE_LOCATION}"
                                        
                                        # Verify extraction
                                        echo "✓ Restore completed successfully"
                                        echo "Extracted files in ${RESTORE_LOCATION}:"
                                        ls -la "${RESTORE_LOCATION}/"
                                        echo "Total files: \$(find "${RESTORE_LOCATION}" -type f | wc -l)"
                                    """
                                    
                                    currentBuild.description = "Restored: ${env.BACKUP_PATH}"
                                }
                            }
                        }
                    }
                }
            }
        }
        
        stage('Post Operation Summary') {
            steps {
                script {
                    if (env.ACTION == 'backup') {
                        echo "=== Backup Summary ==="
                        sh """
                            echo "Backup file: ${env.BACKUP_PATH}"
                            ls -lh "${env.BACKUP_PATH}"
                            echo ""
                            echo "All backups in ${BACKUP_LOCATION}:"
                            ls -lh "${BACKUP_LOCATION}" | head -10
                        """
                    } else {
                        echo "=== Restore Summary ==="
                        sh """
                            echo "Restored from: ${env.BACKUP_PATH}"
                            echo "Restored to: ${RESTORE_LOCATION}"
                            echo ""
                            echo "Restored contents:"
                            find "${RESTORE_LOCATION}" -type f | head -20
                        """
                    }
                }
            }
        }
    }
    
    post {
        success {
            script {
                echo "Operation completed successfully!"
                echo "Action performed: ${env.ACTION}"
                echo "Backup file: ${env.BACKUP_PATH}"
                
                if (env.ACTION == 'backup') {
                    echo "New backup created at: ${env.BACKUP_PATH}"
                } else {
                    echo "Files restored to: ${RESTORE_LOCATION}"
                }
            }
        }
        
        failure {
            script {
                echo "Pipeline failed!"
                echo "Please check the logs for details."
            }
        }
        
        always {
            echo "Cleaning up workspace..."
            cleanWs()
            echo "Pipeline execution completed."
        }
    }
}
