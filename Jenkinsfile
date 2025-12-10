pipeline {
    agent any
    
    parameters {
        file(name: 'UPLOADED_BACKUP_FILE', description: 'Upload PostgreSQL backup tar file')
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
                        mkdir -p "\${BACKUP_LOCATION}" "\${RESTORE_LOCATION}"
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
                    echo "📦 Extracting: ${params.UPLOADED_BACKUP_FILE}"
                    
                    if [[ "${params.UPLOADED_BACKUP_FILE}" == *.tar.gz ]] || [[ "${params.UPLOADED_BACKUP_FILE}" == *.tgz ]]; then
                        tar -xzf "${params.UPLOADED_BACKUP_FILE}" -C "\${RESTORE_LOCATION}"
                    elif [[ "${params.UPLOADED_BACKUP_FILE}" == *.tar ]]; then
                        tar -xf "${params.UPLOADED_BACKUP_FILE}" -C "\${RESTORE_LOCATION}"
                    else
                        echo "❌ Use .tar.gz or .tar"
                        exit 1
                    fi
                    
                    echo "✅ Extracted:"
                    ls -la "\${RESTORE_LOCATION}/"
                """
            }
        }
        
        stage('PostgreSQL Restore') {
            steps {
                sh """
                    echo "🔄 Restoring to ${params.DB_NAME}"
                    
                    # Drop & recreate DB
                    dropdb --if-exists "${params.DB_NAME}" -h "${params.DB_HOST}" -U "${params.DB_USER}" || true
                    createdb "${params.DB_NAME}" -h "${params.DB_HOST}" -U "${params.DB_USER}"
                    
                    # Find restore files (FIXED SYNTAX)
                    RESTORE_FILES=""
                    if [ -f "\${RESTORE_LOCATION}/dump.sql" ]; then
                        RESTORE_FILES="\${RESTORE_LOCATION}/dump.sql"
                    elif [ -f "\${RESTORE_LOCATION}/${params.DB_NAME}.sql" ]; then
                        RESTORE_FILES="\${RESTORE_LOCATION}/${params.DB_NAME}.sql"
                    elif ls "\${RESTORE_LOCATION}"/*.sql 2>/dev/null | grep -q .; then
                        RESTORE_FILES=\$(ls "\${RESTORE_LOCATION}"/*.sql 2>/dev/null | head -1)
                    elif ls "\${RESTORE_LOCATION}"/*.dump 2>/dev/null | grep -q .; then
                        RESTORE_FILES=\$(ls "\${RESTORE_LOCATION}"/*.dump 2>/dev/null | head -1)
                    fi
                    
                    if [ -n "\$RESTORE_FILES" ]; then
                        echo "📊 SQL Restore: \$RESTORE_FILES"
                        psql -h "${params.DB_HOST}" -U "${params.DB_USER}" -d "${params.DB_NAME}" -f "\$RESTORE_FILES"
                    else
                        # pg_dump binary format
                        DUMP_FILE=\$(ls "\${RESTORE_LOCATION}"/*.dump "\${RESTORE_LOCATION}"/*.backup 2>/dev/null 2>&1 | head -1)
                        if [ -n "\$DUMP_FILE" ]; then
                            echo "📦 pg_restore: \$DUMP_FILE"
                            pg_restore --verbose --clean --no-acl --no-owner \\
                                -h "${params.DB_HOST}" -U "${params.DB_USER}" -d "${params.DB_NAME}" "\$DUMP_FILE"
                        else
                            echo "❌ No PostgreSQL files found!"
                            find "\${RESTORE_LOCATION}" -type f
                            exit 1
                        fi
                    fi
                    
                    echo "🎉 Restore completed!"
                """
            }
        }
        
        stage('Verification') {
            steps {
                sh """
                    echo "🔍 Verifying ${params.DB_NAME}..."
                    psql -h "${params.DB_HOST}" -U "${params.DB_USER}" -d "${params.DB_NAME}" \\
                        -c "SELECT count(*) FROM information_schema.tables WHERE table_schema = 'public';" -t -A
                """
            }
        }
    }
    
    post {
        success {
            currentBuild.description = "PostgreSQL restored: ${params.DB_NAME}"
            echo "✅ Database restored: ${params.DB_NAME}@${params.DB_HOST}"
        }
        failure {
            echo "❌ Restore failed!"
        }
    }
}
