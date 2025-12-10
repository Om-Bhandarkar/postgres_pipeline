pipeline {
    agent any

    environment {
        PG_HOST = 'localhost'
        PG_DB   = 'restore_db'
    }

    stages {

        stage('Check Dump File Exists') {
            steps {
                script {
                    echo "Checking dump.sql in workspace..."

                    sh """
                        if [ ! -f dump.sql ]; then
                            echo '❌ ERROR: dump.sql not found in workspace. Please copy the file to the Jenkins workspace.'
                            exit 1
                        fi
                    """

                    echo "✔ dump.sql found!"
                }
            }
        }

        stage('Restore PostgreSQL') {
            environment {
                DB_CREDS = credentials('a5bde45d-3b6d-495d-a022-14f7f3f977ba')   // ← तुझा credentials ID
            }

            steps {
                script {
                    echo "Starting restore..."

                    sh """
                        export PGPASSWORD="${DB_CREDS_PSW}"
                        psql -h ${PG_HOST} -U ${DB_CREDS_USR} -d ${PG_DB} -f dump.sql
                    """

                    echo "✔ Database restore completed!"
                }
            }
        }

        stage('Verify Restore') {
            environment {
                DB_CREDS = credentials('a5bde45d-3b6d-495d-a022-14f7f3f977ba')
            }

            steps {
                script {
                    echo "Verifying tables in database..."

                    sh """
                        export PGPASSWORD="${DB_CREDS_PSW}"
                        psql -h ${PG_HOST} -U ${DB_CREDS_USR} -d ${PG_DB} -c "\\dt"
                    """

                    echo "✔ Verification complete!"
                }
            }
        }
    }

    post {
        success {
            echo "🎉 Pipeline completed successfully!"
        }
        failure {
            echo "❌ Pipeline failed. Check logs!"
        }
    }
}
