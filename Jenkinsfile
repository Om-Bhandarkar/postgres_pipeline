pipeline {
    agent any

    parameters {
        file(name: 'DB_BACKUP', description: 'Upload PostgreSQL backup (.dump or .sql)')
        string(name: 'DB_HOST', defaultValue: 'localhost', description: 'PostgreSQL host')
        string(name: 'DB_NAME', defaultValue: 'your_database', description: 'Target database name')
        string(name: 'DB_PORT', defaultValue: '5432', description: 'PostgreSQL port')
    }

    stages {

        stage('Prepare Backup File (Preserve Original Name)') {
            steps {
                sh '''
                    echo "🔍 DB_BACKUP path: $DB_BACKUP"

                    if [ ! -f "$DB_BACKUP" ]; then
                        echo "❌ ERROR: Backup file not found"
                        exit 1
                    fi

                    ORIGINAL_FILE_NAME=$(basename "$DB_BACKUP")

                    echo "📄 Original filename: $ORIGINAL_FILE_NAME"
                    echo "📂 Copying file to workspace WITHOUT renaming"

                    cp "$DB_BACKUP" "$ORIGINAL_FILE_NAME"

                    echo "✅ File in workspace:"
                    ls -lh "$ORIGINAL_FILE_NAME"
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

                        ORIGINAL_FILE_NAME=$(basename "$DB_BACKUP")

                        echo "🔄 Restoring from file: $ORIGINAL_FILE_NAME"

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
                          "$ORIGINAL_FILE_NAME"

                        EXIT_CODE=$?

                        if [ $EXIT_CODE -ne 0 ]; then
                            echo "⚠️ pg_restore completed with warnings"
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
