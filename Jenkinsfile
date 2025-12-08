pipeline {
    agent any

    parameters {
        file(name: 'BACKUP_FILE', description: 'Upload Postgres backup (.dump or .sql)')
        choice(name: 'BACKUP_FORMAT', choices: ['CUSTOM', 'PLAIN_SQL'], description: 'Backup file format')
    }

    stages {

        stage('Check PostgreSQL Installation') {
            steps {
                script {
                    echo "Checking if PostgreSQL client is installed..."

                    def status = sh(
                        script: "command -v psql >/dev/null 2>&1",
                        returnStatus: true
                    )

                    if (status == 0) {
                        echo "PostgreSQL client already installed ✔️"
                    } else {
                        echo "PostgreSQL NOT installed. Installing using docker-compose…"

                        sh '''
                            if [ ! -f docker-compose.yml ]; then
                                echo "docker-compose.yml missing!"
                                exit 1
                            fi

                            docker compose up -d
                        '''

                        echo "Waiting for PostgreSQL to start..."
                        sleep 12
                    }
                }
            }
        }

        stage('Health Check') {
            steps {
                script {
                    echo "Checking if PostgreSQL service is alive INSIDE container..."

                    sh '''
                        for i in {1..20}; do
                            echo "Attempt $i: checking PostgreSQL inside container..."
                            docker exec postgres_local pg_isready -U admin -d mydb && exit 0
                            sleep 3
                        done

                        echo "PostgreSQL is not responding!"
                        exit 1
                    '''
                }
            }
        }

        stage('Restore Database') {
            steps {
                script {
                    echo "Starting restore process..."
                    echo "Backup file: ${params.BACKUP_FILE}"

                    if (params.BACKUP_FILE == null || params.BACKUP_FILE.trim() == "") {
                        error "❌ No backup file uploaded!"
                    }

                    if (!fileExists("${WORKSPACE}/${params.BACKUP_FILE}")) {
                        error "❌ Backup file not found in workspace: ${WORKSPACE}/${params.BACKUP_FILE}"
                    }

                    echo "Copying backup file into PostgreSQL container..."
                    sh """
                        docker cp "${WORKSPACE}/${params.BACKUP_FILE}" postgres_local:/tmp/backup_file
                    """

                    if (params.BACKUP_FORMAT == "CUSTOM") {
                        echo "Restoring using pg_restore (custom format)..."

                        sh '''
                            docker exec postgres_local bash -c \
                              "pg_restore --clean --if-exists --no-owner -U admin -d mydb /tmp/backup_file"
                        '''
                    } else {
                        echo "Restoring using psql (plain SQL)..."

                        sh '''
                            docker exec postgres_local bash -c \
                              "psql -U admin -d mydb -f /tmp/backup_file"
                        '''
                    }

                    echo "Restore completed ✔️"
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
