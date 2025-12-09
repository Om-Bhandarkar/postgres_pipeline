pipeline {
    agent any
 
    parameters {
        file(name: 'BACKUP_FILE', description: 'Upload Postgres backup (.dump or .sql)')
        choice(name: 'BACKUP_FORMAT', choices: ['CUSTOM', 'PLAIN_SQL'], description: 'Backup file format')
    }
 
    stages {
        stage('Setup') {
            steps {
                script {
                    if (params.BACKUP_FILE == null) {
                        error "Please use 'Build with Parameters' and upload a backup file first!"
                    }
                    
                    // Start PostgreSQL
                    sh '''
                        docker stop postgres_local || true
                        docker rm postgres_local || true
                        
                        docker run -d \
                          --name postgres_local \
                          -e POSTGRES_USER=admin \
                          -e POSTGRES_PASSWORD=admin123 \
                          -e POSTGRES_DB=mydb \
                          -p 5432:5432 \
                          postgres:15
                        
                        sleep 10
                    '''
                }
            }
        }
        
        stage('Restore') {
            steps {
                script {
                    // Find the uploaded file
                    sh '''
                        echo "Looking for uploaded file..."
                        find . -type f -name "*.dump" -o -name "*.sql" -o -name "*.backup" | head -5
                        echo ""
                        ls -la
                    '''
                    
                    // Get the first backup file found
                    def backupFile = sh(script: 'find . -type f \\( -name "*.dump" -o -name "*.sql" \\) | head -1', returnStdout: true).trim()
                    
                    if (!backupFile) {
                        error "No backup file found! Please upload a .dump or .sql file."
                    }
                    
                    echo "Using backup file: ${backupFile}"
                    
                    sh """
                        docker cp "${backupFile}" postgres_local:/tmp/backup_file
                        
                        if [ "${params.BACKUP_FORMAT}" = "CUSTOM" ]; then
                            docker exec postgres_local pg_restore -U admin -d mydb /tmp/backup_file
                        else
                            docker exec postgres_local psql -U admin -d mydb -f /tmp/backup_file
                        fi
                    """
                }
            }
        }
    }
}
