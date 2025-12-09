pipeline {
    agent any
 
    parameters {
        file(name: 'BACKUP_FILE', description: 'Upload Postgres backup (.dump or .sql)')
        choice(name: 'BACKUP_FORMAT', choices: ['CUSTOM', 'PLAIN_SQL'], description: 'Backup file format')
    }
 
    stages {
 
        stage('Check PostgreSQL Installation') {
            steps {
                script {
                    echo "Checking if PostgreSQL client is installed..."
                    
                    // Check if Docker is running
                    def dockerStatus = sh(
                        script: "command -v docker >/dev/null 2>&1",
                        returnStatus: true
                    )
                    
                    if (dockerStatus != 0) {
                        error "Docker is not installed or not running!"
                    }
 
                    // Check if PostgreSQL container exists and is running
                    def containerStatus = sh(
                        script: "docker ps --filter 'name=postgres_local' --format '{{.Names}}' | grep -q postgres_local",
                        returnStatus: true
                    )
 
                    if (containerStatus == 0) {
                        echo "PostgreSQL container is already running ✔️"
                    } else {
                        echo "Starting PostgreSQL container using docker-compose…"
 
                        sh '''
                            if [ ! -f docker-compose.yml ]; then
                                echo "Creating docker-compose.yml..."
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
                            fi
 
                            docker compose up -d
                        '''
 
                        echo "Waiting for PostgreSQL to start..."
                        sleep 10
                    }
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
                            if docker exec postgres_local pg_isready -U admin -d mydb > /dev/null 2>&1; then
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
                    
                    // Get the actual filename
                    def backupFileName = sh(
                        script: "basename '${params.BACKUP_FILE}'",
                        returnStdout: true
                    ).trim()
                    
                    echo "Backup file: ${backupFileName}"
                    
                    // Copy file to container
                    sh """
                        docker cp "${WORKSPACE}/${backupFileName}" postgres_local:/tmp/backup_file
                    """
                    
                    // Check file exists in container
                    sh '''
                        docker exec postgres_local ls -la /tmp/backup_file
                    '''
 
                    if (params.BACKUP_FORMAT == "CUSTOM") {
                        echo "Restoring using pg_restore (custom format)..."
                        
                        sh '''
                            docker exec postgres_local bash -c \
                              "pg_restore --verbose --clean --if-exists --no-owner -U admin -d mydb /tmp/backup_file"
                        '''
                    } else {
                        echo "Restoring using psql (plain SQL)..."
                        
                        sh '''
                            docker exec postgres_local bash -c \
                              "psql -U admin -d mydb -f /tmp/backup_file 2>&1 | grep -v 'already exists'"
                        '''
                    }
                    
                    // Verify restore
                    echo "Verifying restore..."
                    sh '''
                        docker exec postgres_local bash -c \
                          "psql -U admin -d mydb -c 'SELECT COUNT(*) as total_tables FROM information_schema.tables WHERE table_schema = \\'public\\';'"
                    '''
                    
                    echo "Restore completed successfully ✔️"
                }
            }
        }
    }
 
    post {
        success {
            echo "Database restored successfully! ✔️"
            echo "You can connect to:"
            echo "Host: localhost"
            echo "Port: 5432"
            echo "Database: mydb"
            echo "Username: admin"
            echo "Password: admin123"
        }
        failure {
            echo "Pipeline failed — check logs ❌"
        }
        always {
            // Cleanup backup file from container
            sh '''
                docker exec postgres_local rm -f /tmp/backup_file 2>/dev/null || true
            '''
        }
    }
}
