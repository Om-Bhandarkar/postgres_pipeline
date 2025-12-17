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
                        credentialsId: 'f6142eb9-730e-4240-9ec9-63be344f8bec',   // Jenkins credential ID
                        usernameVariable: 'DB_USER',        // fetched from Jenkins
                        passwordVariable: 'DB_PASS'         // fetched from Jenkins
                    )
                ]) {
                    sh '''
                        set -e

                        # Jenkins injects DB_USER and DB_PASS securely
                        export PGPASSWORD="$DB_PASS"

                        echo "Restoring database '$DB_NAME' on $DB_HOST:$DB_PORT"
                        echo "Using DB user: $DB_USER"

                        pg_restore \
                          -h "$DB_HOST" \
                          -p "$DB_PORT" \
                          -U "$DB_USER" \
                          -d "$DB_NAME" \
                          --clean \
                          --if-exists \
                          --no-owner \
                          --verbose \
                          "$DB_BACKUP"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo '✅ Database restore successful!'
            cleanWs()
        }
        failure {
            echo '❌ Restore failed. Check logs above.'
        }
    }
}
