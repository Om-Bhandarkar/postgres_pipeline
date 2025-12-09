pipeline {
    agent any
 
    parameters {
        file(name: 'BACKUP_FILE', description: 'Upload Postgres backup (.dump or .sql)')
        choice(name: 'BACKUP_FORMAT', choices: ['CUSTOM', 'PLAIN_SQL'], description: 'Backup file format', defaultValue: 'CUSTOM')
    }
 
    stages {
        stage('Check Parameters') {
            steps {
                script {
                    echo "DEBUG: BACKUP_FILE parameter value: ${params.BACKUP_FILE}"
                    echo "DEBUG: BACKUP_FORMAT parameter value: ${params.BACKUP_FORMAT}"
                    
                    // Check if parameters are provided
                    if (params.BACKUP_FILE == null) {
                        error """
                        ❌ No backup file uploaded!
                        
                        Please use 'Build with Parameters' and upload a backup file.
                        
                        Steps to fix:
                        1. Go to your pipeline in Jenkins
                        2. Click 'Build with Parameters'
                        3. Upload your backup file (.dump or .sql)
                        4. Select backup format
                        5. Click Build
                        """
                    }
                }
            }
        }
 
        stage('Check PostgreSQL Installation') {
            steps {
                script {
                    echo "Checking if PostgreSQL client is installed..."
                    
                    def dockerStatus = sh(
                        script: "command -v docker >/dev/null 2>&1",
                        returnStatus: true
                    )
                    
                    if (dockerStatus != 0) {
                        error "Docker is not installed or not running!"
                    }
 
                    def containerStatus = sh(
                        script: "docker ps --filter 'name=postgres_local' --format '{{.Names}}' | grep -q postgres_local",
                        returnStatus: true
                    )
 
                    if (containerStatus == 0) {
                        echo "PostgreSQL container is already running ✔️"
                        
                        // Restart container to ensure clean state
                        sh '''
                            docker stop postgres_local || true
                            docker rm postgres_local || true
                        '''
                    }
                    
                    echo "Starting PostgreSQL container using docker-compose…"
                    
                    // Always create fresh container
                    sh '''
                        # Create docker-compose.yml if not exists
                        cat > docker-compose.yml << 'EOF'
version: '3.8'
services:
  postgres_local:
    image: postgres:15
    container_name: postgres_local
    restart: always
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: admin123
      POSTGRES_DB: mydb
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
volumes:
  postgres_data:
EOF
 
                        docker compose down || true
                        docker compose up -d
                    '''
 
                    echo "Waiting for PostgreSQL to start..."
                    sleep 10
                }
            }
        }
 
        stage('Health Check') {
            steps {
                script {
                    echo "Checking if PostgreSQL service is alive..."
 
                    sh '''
                        # Wait for PostgreSQL to be ready
                        for i in {1..30}; do
                            echo "Attempt $i/30: checking PostgreSQL..."
                            if docker exec postgres_local pg_isready -U admin > /dev/null 2>&1; then
                                echo "PostgreSQL is ready!"
                                break
                            fi
                            
                            if [ $i -eq 30 ]; then
                                echo "PostgreSQL is not responding after 30 attempts!"
                                exit 1
                            fi
                            
                            sleep 2
                        done
                    '''
                }
            }
        }
 
        stage('Restore Database') {
            steps {
                script {
                    echo "Starting restore process..."
                    
                    // Get the actual filename from uploaded file
                    def backupFile = params.BACKUP_FILE
                    
                    // Check if file exists in workspace
                    sh '''
                        echo "Listing files in workspace:"
                        ls -la ${WORKSPACE}/
                        echo ""
                        echo "Looking for backup files:"
                        find ${WORKSPACE} -name "*.dump" -o -name "*.sql" -o -name "*.backup" | head -10
                    '''
                    
                    // Find the uploaded file
                    def backupFileName = sh(
                        script: '''
                            find ${WORKSPACE} -type f \( -name "*.dump" -o -name "*.sql" -o -name "*.backup" \) -newer /tmp/dummy 2>/dev/null | head -1
                        ''',
                        returnStdout: true
                    ).trim()
                    
                    if (!backupFileName) {
                        backupFileName = sh(
                            script: '''
                                find ${WORKSPACE} -type f | grep -i "backup\|dump\|sql" | head -1
                            ''',
                            returnStdout: true
                        ).trim()
                    }
                    
                    if (!backupFileName) {
                        error """
                        ❌ No backup file found in workspace!
                        
                        Files in workspace:
                        ${sh(script: 'ls -la ${WORKSPACE}/', returnStdout: true)}
                        
                        Please make sure:
                        1. You use 'Build with Parameters'
                        2. Upload a valid backup file (.dump, .sql, .backup)
                        3. The file is uploaded successfully
                        """
                    }
                    
                    def shortName = sh(
                        script: "basename '${backupFileName}'",
                        returnStdout: true
                    ).trim()
                    
                    echo "Using backup file: ${shortName}"
                    echo "Full path: ${backupFileName}"
                    
                    // Copy file to container
                    sh """
                        echo "Copying ${backupFileName} to container..."
                        docker cp "${backupFileName}" postgres_local:/tmp/backup_file
                    """
                    
                    // Verify file in container
                    sh '''
                        echo "File details in container:"
                        docker exec postgres_local ls -la /tmp/backup_file
                        echo ""
                        echo "File size:"
                        docker exec postgres_local wc -c /tmp/backup_file
                    '''
 
                    echo "Restoring database 'mydb'..."
                    
                    // First, drop and recreate database
                    sh '''
                        docker exec postgres_local bash -c \
                          "psql -U admin -c 'DROP DATABASE IF EXISTS mydb;'"
                        docker exec postgres_local bash -c \
                          "psql -U admin -c 'CREATE DATABASE mydb;'"
                    '''
                    
                    if (params.BACKUP_FORMAT == "CUSTOM") {
                        echo "Restoring using pg_restore (custom format)..."
                        
                        sh '''
                            docker exec postgres_local bash -c \
                              "pg_restore --verbose --clean --if-exists --no-owner -U admin -d mydb /tmp/backup_file 2>&1 | tail -20"
                        '''
                    } else {
                        echo "Restoring using psql (plain SQL)..."
                        
                        sh '''
                            docker exec postgres_local bash -c \
                              "psql -U admin -d mydb -f /tmp/backup_file 2>&1 | grep -E '(ERROR|FATAL|already exists)' || true"
                        '''
                    }
                    
                    // Verify restore
                    echo "Verifying restore..."
                    sh '''
                        echo "=== Database Verification ==="
                        docker exec postgres_local bash -c \
                          "psql -U admin -d mydb -c 'SELECT COUNT(*) as total_tables FROM information_schema.tables WHERE table_schema = \\'public\\';'"
                        echo ""
                        echo "=== Top 5 tables ==="
                        docker exec postgres_local bash -c \
                          "psql -U admin -d mydb -c 'SELECT table_name FROM information_schema.tables WHERE table_schema = \\'public\\' LIMIT 5;'"
                    '''
                    
                    echo "Restore completed successfully ✔️"
                }
            }
        }
    }
 
    post {
        success {
            echo """
            ========================================
            Database restored successfully! ✔️
            
            Connection details:
            Host: localhost
            Port: 5432
            Database: mydb
            Username: admin
            Password: admin123
            
            Connect using:
            psql -h localhost -p 5432 -U admin -d mydb
            ========================================
            """
        }
        failure {
            echo """
            ========================================
            Pipeline failed! ❌
            
            Common issues:
            1. Did you use 'Build with Parameters'?
            2. Did you upload a valid backup file?
            3. Check the backup file format matches selection
            
            To retry:
            1. Click 'Build with Parameters'
            2. Upload backup file
            3. Select correct format
            4. Click Build
            ========================================
            """
        }
        always {
            sh '''
                echo "Cleaning up..."
                docker exec postgres_local rm -f /tmp/backup_file 2>/dev/null || true
                echo "Cleanup done!"
            '''
        }
    }
}
