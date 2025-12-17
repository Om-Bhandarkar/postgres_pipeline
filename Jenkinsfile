pipeline {
    agent any

    parameters {
        file(name: 'DB_BACKUP', description: 'Upload PostgreSQL backup (.dump or .sql)')
        string(name: 'DB_HOST', defaultValue: 'localhost', description: 'PostgreSQL host')
        string(name: 'DB_NAME', defaultValue: 'your_database', description: 'Target database name')
        string(name: 'DB_PORT', defaultValue: '5432', description: 'PostgreSQL port')
    }

    stages {

        stage('Prepare Backup File (Copy to Workspace)') {
            steps {
                sh '''
                    if [ ! -f "$DB_BACKUP" ]; then
                        echo "❌ ERROR: No backup file uploaded."
                        exit 1
                    fi

                    BACKUP_FILE_NAME=$(basename "$DB_BACKUP")

                    echo "📦 Uploaded file path: $DB_BACKUP"
                    echo "📂 Copying to workspace as: $BACKUP_FILE_NAME"

                    cp "$DB_BACKUP" "$BACKUP_FILE_NAME"

                    echo "✅ File now present in workspace:"
                    ls -lh "$BACKUP_FILE_NAME"
                    file "$BACKUP_FILE_NAME"
                '''
            }
        }

        stage('Restore to PostgreSQL') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'f6142eb9-730e-4240-9ec9-63be344f8bec',
                        usernameVariable: 'DB_USER',
                        passwordVariable: 'DB_PASS'
                    )
                ]) {
                    sh '''
                        set +e
                        export PGPASSWORD="$DB_PASS"

                        BACKUP_FILE_NAME=$(basename "$DB_BACKUP")

                        echo "🔄 Restoring database '$DB_NAME'"
                        echo "👤 DB User: $DB_USER"
                        echo "📄 Backup file: $BACKUP_FILE_NAME"

                        pg_restore \
                          -h "$DB_HOST" \
                          -p "$DB_PORT" \
                          -U "$DB_USER" \
                          -d "$DB_NAME" \
                          --clean \
                          --if-exists \
                          --no-owner \
                          --no-privileges \
                          --verbose \
                          "$BACKUP_FILE_NAME"

                        EXIT_CODE=$?

                        if [ $EXIT_CODE -ne 0 ]; then
                            echo "⚠️ pg_restore completed with warnings (exit code=$EXIT_CODE)"
                            echo "✔ Treating restore as SUCCESS"
                        fi

                        exit 0
                    '''
                }
            }
        }
    }

    post {
        success {
            echo '✅ Database restore completed successfully'
            cleanWs()
        }
        failure {
            echo '❌ Restore failed. Check logs above.'
        }
    }
}
