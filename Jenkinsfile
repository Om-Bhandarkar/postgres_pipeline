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
                    echo "Checking QA15Sep25 in workspace..."

                    sh """
                        if [ ! -f QA15Sep25 ]; then
                            echo '❌ ERROR: QA15Sep25 file not found in workspace.'
                            exit 1
                        fi
                    """

                    echo "✔ QA15Sep25 found!"
                }
            }
        }

        stage('Restore PostgreSQL') {
            environment {
                DB_CREDS = credentials('a5bde45d-3b6d-495d-a022-14f7f3f977ba')
            }

            steps {
                script {
                    echo "Restoring database using QA15Sep25..."

                    sh """
                        export PGPASSWORD="${DB_CREDS_PSW}"
                        psql -h ${PG_HOST} -U ${DB_CREDS_USR} -d ${PG_DB} -f QA15Sep25
                    """

                    echo "✔ Restore complete"
                }
            }
        }

        stage('Verify Restore') {
            environment {
                DB_CREDS = credentials('a5bde45d-3b6d-495d-a022-14f7f3f977ba')
            }

            steps {
                script {
                    echo "Verifying restore..."

                    sh """
                        export PGPASSWORD="${DB_CREDS_PSW}"
                        psql -h ${PG_HOST} -U ${DB_CREDS_USR} -d ${PG_DB} -c "\\dt"
                    """
                }
            }
        }
    }

    post {
        success { echo "🎉 Pipeline completed successfully!" }
        failure { echo "❌ Pipeline failed. Check logs!" }
    }
}
