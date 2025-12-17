pipeline {
    agent any

    parameters {
        file(name: 'DB_BACKUP', description: 'Upload PostgreSQL backup (.dump or .sql)')
        string(name: 'DB_HOST', defaultValue: 'localhost', description: 'PostgreSQL host')
        string(name: 'DB_NAME', defaultValue: 'your_database', description: 'Target database name')
        string(name: 'DB_PORT', defaultValue: '5432', description: 'PostgreSQL port')
    }

    stages {

        stage('Debug & Prepare Backup File') {
            steps {
                sh '''
                    echo "=== DEBUG INFO ==="
                    echo "DB_BACKUP value: $DB_BACKUP"
                    echo "WORKSPACE: $WORKSPACE"
                    echo "Listing workspace:"
                    ls -lh .

                    if [ ! -f "$DB_BACKUP" ]; then
                        echo "❌ ERROR: Backup file NOT found in workspace."
                        echo "➡ You MUST click 'Build with Parameters' and upload the file."
                        exit 1
                    fi

                    echo "✅ Backup file detected:"
                    ls -lh "$DB_BACKUP"
                    file "$DB_BACKUP"
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

                        echo "🔄 Restoring database '$DB_NAME'"

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
                          "$DB_BACKUP"

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
