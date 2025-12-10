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

        stage('Copy Dump File') {
            steps {
                script {
                    echo "Uploaded file path: ${params.DUMP_FILE}"
        
                    sh """
                        cp "${params.DUMP_FILE}" dump.sql
                    """
        
                    echo "File copied."
                }
            }
        }


        stage('Restore to PostgreSQL') {
            environment {
                DB_CREDS = credentials('a5bde45d-3b6d-495d-a022-14f7f3f977ba')  
            }

            steps {
                echo "Restoring database..."

                sh """
                    export PGPASSWORD="${DB_CREDS_PSW}"
                    psql -h ${PG_HOST} -U ${DB_CREDS_USR} -d ${PG_DB} -f dump.sql
                """

                echo "Database restore completed."
            }
        }
    }

    post {
        success { echo "Pipeline completed successfully!" }
        failure { echo "Pipeline failed!" }
    }
}
