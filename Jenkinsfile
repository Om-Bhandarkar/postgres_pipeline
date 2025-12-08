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

                    echo "Waiting for container to start..."
                }
            }
        }

        stage('Health Check INSIDE Container') {
            steps {
                script {
                    echo "Checking if PostgreSQL is ready..."

                    sh '''
                        for i in {1..20}; do
                            echo "Attempt $i: checking Postgres inside container..."
                            docker exec postgres_local pg_isready -U admin -d mydb && exit 0
                            sleep 3
                        done

                        echo "PostgreSQL did not become ready!"
                        exit 1
                    '''
                }
            }
        }

        stage('Restore Database') {
            steps {
                script {
                    echo "Copying backup file into container..."

                    sh """
                        docker cp "${WORKSPACE}/${params.BACKUP_FILE}" postgres_local:/tmp/backup_file

                    """

                    if (params.BACKUP_FORMAT == 'CUSTOM') {
                        echo "Restoring custom dump using pg_restore..."

                        sh """
                            docker exec postgres_local bash -c "pg_restore --clean --if-exists --no-owner -U admin -d mydb /tmp/backup_file"
                        """

                    } else {
                        echo "Restoring SQL file using psql..."

                        sh """
                            docker exec postgres_local bash -c "psql -U admin -d mydb -f /tmp/backup_file"
                        """
                    }

                    echo "Restore completed successfully!"
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline finished successfully — DB restored ✔️"
        }
        failure {
            echo "Pipeline failed — check logs ❌"
        }
    }
}
