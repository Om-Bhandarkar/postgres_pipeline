pipeline {
    agent any

    parameters {
        file(name: 'DUMP_FILE', description: 'Upload your PostgreSQL dump file')
    }

    environment {
        PG_HOST = 'localhost'
        PG_DB   = 'restore_db'
    }

    stages {

        stage('Show Uploaded File Path') {
            steps {
                script {
                    echo "Jenkins uploaded file path = ${params.DUMP_FILE}"

                    if (!params.DUMP_FILE) {
                        error "❌ ERROR: No dump file uploaded. Please upload a .sql or .dump file."
                    }
                }
            }
        }

        stage('Copy Dump File') {
            steps {
                script {
                    echo "Copying uploaded dump file to workspace..."

                    // Correct and safe copy (works in all Jenkins versions)
                    sh """
                        cp "${params.DUMP_FILE}" ./dump.sql
                    """

                    echo "✔ File copied as dump.sql"
                }
            }
        }

        stage('Restore to PostgreSQL') {
            environment {
                DB_CREDS = credentials('a5bde45d-3b6d-495d-a022-14f7f3f977ba')  
            }

            steps {
                script {
                    echo "Restoring database..."

                    sh """
                        export PGPASSWORD="${DB_CREDS_PSW}"
                        psql -h ${PG_HOST} -U ${DB_CREDS_USR} -d ${PG_DB} -f dump.sql
                    """

                    echo "✔ Database restore completed."
                }
            }
        }

        stage('Verify Restore') {
            environment {
                DB_CREDS = credentials('a5bde45d-3b6d-495d-a022-14f7f3f977ba')
            }

            steps {
                script {
                    echo "Verifying DB restore..."

                    sh """
                        export PGPASSWORD="${DB_CREDS_PSW}"
                        psql -h ${PG_HOST} -U ${DB_CREDS_USR} -d ${PG_DB} -c "\\dt"
                    """
                }
            }
        }
    }

    post {
        success {
            echo "🎉 Pipeline completed successfully!"
        }
        failure {
            echo "❌ Pipeline failed!"
        }
    }
}
