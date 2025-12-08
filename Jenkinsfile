pipeline {
    agent any

    parameters {
        file(name: 'BACKUP_FILE', description: 'Upload Postgres backup (.dump or .sql)')
        choice(name: 'BACKUP_FORMAT', choices: ['CUSTOM', 'PLAIN_SQL'], description: 'Backup file format')
    }

    stages {

        stage('Start PostgreSQL Container') {
            steps {
                script {
                    echo "Starting PostgreSQL using docker-compose…"

                    sh '''
                        if [ ! -f docker-compose.yml ]; then
                            echo "docker-compose.yml missing!"
                            exit 1
                        fi

                        docker compose up -d
                    '''

                    echo "Waiting for PostgreSQL to start..."
                }
            }
        }

        stage('Health Check Inside Container') {
            steps {
                script {
                    echo "Checking if PostgreSQL service is alive inside container..."

                    sh '''
                        for i in {1..15}; do
                            echo "Attempt $i..."
                            docker exec postgres_local pg_isready -U admin -d mydb && exit 0
                            sleep 3
                        done

                        echo "PostgreSQL is not responding inside container!"
                        exit 1
                    '''
                }
            }
        }

        stage('Restore Database') {
            steps {
                script {

                    echo "Starting restore process..."
                    echo "Backup file path: ${BACKUP_FILE}"

                    // Copy backup file into container
                    sh """
                        docker cp ${BACKUP_FILE} postgres_local:/tmp/backup_file
                    """

                    if (params.BACKUP_FORMAT == "CUSTOM") {
                        echo "Restoring using pg_restore (custom format)..."

                        sh """
                            docker exec postgres_local bash -c "pg_restore --clean --if-exists --no-owner -U admin -d mydb /tmp/backup_file"
                        """

                    } else {
                        echo "Restoring using psql (plain SQL)..."

                        sh """
                            docker exec postgres_local bash -c "psql -U admin -d mydb -f /tmp/backup_file"
                        """
                    }

                    echo "Restore completed successfully ✔️"
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline finished successfully — database restored ✔️"
        }
        failure {
            echo "Pipeline failed — check logs ❌"
        }
    }
}
