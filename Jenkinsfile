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
                    echo "Checking if PostgreSQL service is alive..."

                    def result = sh(
                        script: "PGPASSWORD=admin123 psql -U admin -h localhost -d mydb -c '\\l' >/dev/null 2>&1",
                        returnStatus: true
                    )

                    if (result == 0) {
                        echo "PostgreSQL is up and responding ✔️"
                    } else {
                        error("PostgreSQL is not responding ❌")
                    }
                }
            }
        }

        stage('Restore Database') {
            steps {
                script {
                    echo "Starting restore process..."
                    echo "Backup file path: ${params.BACKUP_FILE}"

                    if (!fileExists("${WORKSPACE}/${BACKUP_FILE}")) {
                        error("Backup file not found in workspace!")
                    }

                    if (params.BACKUP_FORMAT == "CUSTOM") {
                        echo "Restoring using pg_restore (custom format)..."

                        sh """
                            export PGPASSWORD=admin123
                            pg_restore --clean --if-exists --no-owner \
                                -h localhost -p 5432 -U admin -d mydb \
                                ${BACKUP_FILE}
                        """

                    } else {
                        echo "Restoring using psql (plain SQL)..."

                        sh """
                            export PGPASSWORD=admin123
                            psql -h localhost -p 5432 -U admin -d mydb \
                                -f ${BACKUP_FILE}
                        """
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
this is my jenkinsfile
