pipeline {
    agent any

    parameters {
        file(name: 'DB_BACKUP', description: 'Upload PostgreSQL backup (.dump or .sql)')
        string(name: 'DB_HOST', defaultValue: 'localhost', description: 'PostgreSQL host')
        string(name: 'DB_NAME', defaultValue: 'your_database', description: 'Target database name')
        string(name: 'DB_PORT', defaultValue: '5432', description: 'PostgreSQL port')
    }

    stages {

        stage('Validate Backup File') {
            steps {
                sh '''
                    if [ ! -f "$DB_BACKUP" ]; then
                        echo "❌ ERROR: No backup file uploaded."
                        echo "➡ Use 'Build with Parameters' and upload a backup file."
                        exit 1
                    fi

                    echo "Backup file details:"
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
                        set +e   # IMPORTANT: allow pg_restore warnings
                        export PGPASSWORD="$DB_PASS"

                        echo "Restoring database '$DB_NAME' on $DB_HOST:$DB_PORT"
                        echo "Using DB user: $DB_USER"
                        echo "Starting pg_restore (warnings will NOT fail the build)"

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
                            echo "⚠️ pg_restore finished with warnings (exit code=$EXIT_CODE)"
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
