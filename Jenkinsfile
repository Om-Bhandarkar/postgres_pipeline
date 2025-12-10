pipeline {
    agent any
    
    parameters {
        file(
            name: '',
            description: 'Upload PostgreSQL backup tar file'
        )
        string(name: 'DB_NAME', defaultValue: 'your_db', description: 'Database name to restore')
        string(name: 'DB_USER', defaultValue: 'postgres', description: 'Database user')
        string(name: 'DB_HOST', defaultValue: 'localhost', description: 'Database host')
        password(name: 'DB_PASSWORD', description: 'Database password')
    }
    
    environment {
        RESTORE_LOCATION = "${WORKSPACE}/restored_files/"
        BACKUP_LOCATION = "${WORKSPACE}/backups/"
        TIMESTAMP = sh(script: 'date +%Y%m%d_%H%M%S', returnStdout: true).trim()
        PGPASSWORD = "${params.DB_PASSWORD}"
    }
    
    stages {
        stage('Initialize') {
            steps {
                script {
                    echo "Starting PostgreSQL restore pipeline..."
                    sh """
                        mkdir -p "${BACKUP_LOCATION}" "${RESTORE_LOCATION}"
                        echo "Workspace: ${WORKSPACE}"
                    """
                    
                    if (!params.UPLOADED_BACKUP_FILE) {
                        error "❌ Please upload a backup tar file!"
                    }
                    env.ACTION = 'postgres_restore'
                }
            }
        }
        
        stage('Extract Backup Tar') {
            steps {
                sh """
                    echo "📦 Extracting uploaded backup: ${params.UPLOADED_BACKUP_FILE}"
                    
                    # Extract tar file (supports tar.gz, tar, tgz)
                    if [[ "${params.UPLOADED_BACKUP_FILE}" == *.tar.gz ]] || [[ "${params.UPLOADED_BACKUP_FILE}" == *.tgz ]]; then
                        tar -xzf "${params.UPLOADED_BACKUP_FILE}" -C "${RESTORE_LOCATION}"
                    elif [[ "${params.UPLOADED_BACKUP_FILE}" == *.tar ]]; then
                        tar -xf "${params.UPLOADED_BACKUP_FILE}" -C "${RESTORE_LOCATION}"
                    else
                        echo "❌ Unsupported file format. Use .tar.gz or .tar"
                        exit 1
                    fi
                    
                    echo "✅ Extracted contents:"
                    ls -la "${RESTORE_LOCATION}/"
                    find "${RESTORE_LOCATION}" -name "*.sql" -o -name "*.dump" | head -10
                """
            }
        }
        
        stage('PostgreSQL Restore') {
            steps {
                sh """
                    echo "🔄 Starting PostgreSQL restore to ${params.DB_NAME}"
                    
                    # Drop and recreate database
                    dropdb --if-exists "${params.DB_NAME}" -h "${params.DB_HOST}" -U "${params.DB_USER}" || true
                    createdb "${params.DB_NAME}" -h "${params.DB_HOST}" -U "${params.DB_USER}"
                    
                    echo "✅ Database ${params.DB_NAME} created"
                    
                    # Find and restore SQL dump files
                    RESTORE_FILES=""
                    if [ -f "${RESTORE_LOCATION}/dump.sql" ]; then
                        RESTORE_FILES="${RESTORE_LOCATION}/dump.sql"
                    elif [ -f "${RESTORE_LOCATION}/${params.DB_NAME}.sql" ]; then
                        RESTORE_FILES="${RESTORE_LOCATION}/${params.DB_NAME}.sql"
                    elif ls "${RESTORE_LOCATION}"/*.sql >/dev/null 2>&1; then
                        RESTORE_FILES=$(ls "${RESTORE_LOCATION}"/*.sql | head -1)
                    elif ls "${RESTORE_LOCATION}"/*.dump >/dev/null 2>&1; then
                        RESTORE_FILES=$(ls "${RESTORE_LOCATION}"/*.dump | head -1)
                    fi
                    
                    if [ -n "\${RESTORE_FILES}" ]; then
                        echo "📊 Restoring from: \${RESTORE_FILES}"
                        psql -h "${params.DB_HOST}" -U "${params.DB_USER}" -d "${params.DB_NAME}" -f "\${RESTORE_FILES}"
                        echo "✅ SQL restore completed"
                    else
                        echo "🔍 Searching for pg_dump format files..."
                        
                        # Handle pg_dump binary format
                        if ls "${RESTORE_LOCATION}"/*.dump >/dev/null 2>&1 || ls "${RESTORE_LOCATION}"/*.backup >/dev/null 2>&1; then
                            DUMP_FILE=$(ls "${RESTORE_LOCATION}"/*.dump "${RESTORE_LOCATION}"/*.backup 2>/dev/null | head -1)
                            echo "📦 Restoring pg_dump binary: \${DUMP_FILE}"
                            pg_restore --verbose --clean --no-acl --no-owner -h "${params.DB_HOST}" -U "${params.DB_USER}" -d "${params.DB_NAME}" "\${DUMP_FILE}"
                            echo "✅ pg_dump restore completed"
                        else
                            echo "❌ No valid PostgreSQL dump files found!"
                            echo "Available files:"
                            find "${RESTORE_LOCATION}" -type f
                            exit 1
                        fi
                    fi
                    
                    echo "🎉 PostgreSQL restore completed successfully!"
                """
            }
        }
        
        stage('Verification') {
            steps {
                sh """
                    echo "🔍 Verifying database restoration..."
                    
                    # Check database exists and has tables
                    psql -h "${params.DB_HOST}" -U "${params.DB_USER}" -d "${params.DB_NAME}" -c "SELECT count(*) as table_count FROM information_schema.tables WHERE table_schema = 'public';" -t -A
                    
                    # Show recent tables
                    psql -h "${params.DB_HOST}" -U "${params.DB_USER}" -d "${params.DB_NAME}" -c "
                        SELECT schemaname,tablename FROM pg_tables 
                        WHERE schemaname='public' 
                        ORDER BY tablename DESC 
                        LIMIT 5;
                    " -t -A
                """
            }
        }
        
        stage('Summary') {
            steps {
                script {
                    currentBuild.description = "PostgreSQL restored: ${params.DB_NAME} from ${params.UPLOADED_BACKUP_FILE}"
                    env.RESTORE_DB = params.DB_NAME
                }
                sh """
                    echo "=== POSTGRESQL RESTORE SUMMARY ==="
                    echo "✅ Database: ${params.DB_NAME}"
                    echo "✅ Host: ${params.DB_HOST}"
                    echo "✅ Extracted files: ${RESTORE_LOCATION}"
                    echo "📊 Build artifacts available for download"
                """
            }
        }
    }
    
    post {
        success {
            echo "🎉 PostgreSQL Database Restore COMPLETED!"
            echo "Database: ${params.DB_NAME}@${params.DB_HOST}"
            echo "Download extracted files: ${BUILD_URL}artifact/"
        }
        failure {
            echo "❌ Restore failed! Database may be partially restored."
            echo "Check logs above for errors."
        }
        always {
            sh """
                echo "Cleanup: Keeping extracted files as artifacts"
                echo "Extracted files location: ${RESTORE_LOCATION}"
            """
        }
    }
}
