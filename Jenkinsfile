pipeline {
    agent any

    parameters {
        file(
            name: 'UPLOADED_BACKUP_FILE',
            description: 'Upload a PostgreSQL backup file (.sql / .tar / .gz)'
        )
    }

    environment {
        TEMP_RESTORE_DIR = "/tmp/jenkins_restore_${BUILD_ID}/"
        PGHOST = "127.0.0.1"
        PGUSER = "postgres"
        PGDATABASE = "restore_db"
        PGPASSWORD = "espl@2017"
    }

    stages {

        stage('Handle Uploaded File') {
            steps {
                script {
                    if (!params.UPLOADED_BACKUP_FILE) {
                        error("No backup file uploaded!")
                    }

                    echo "Uploaded file path: ${params.UPLOADED_BACKUP_FILE}"

                    // Create temp restore directory
                    sh "mkdir -p ${TEMP_RESTORE_DIR}"

                    // Copy file parameter into temp directory
                    sh """
                        cp "${params.UPLOADED_BACKUP_FILE}" "${TEMP_RESTORE_DIR}/"
                    """

                    // Store extracted file path
                    env.FILE_BASENAME = sh(
                        script: "basename ${params.UPLOADED_BACKUP_FILE}",
                        returnStdout: true
                    ).trim()

                    env.RESTORE_FILE = "${TEMP_RESTORE_DIR}/${env.FILE_BASENAME}"

                    echo "Copied uploaded file to temp: ${env.RESTORE_FILE}"
                }
            }
        }

        stage('Extract Backup') {
            steps {
                script {
                    sh """
                        FILE="${env.RESTORE_FILE}"

                        if [[ "\$FILE" == *.tar.gz || "\$FILE" == *.tgz ]]; then
                            tar -xzf "\$FILE" -C "${TEMP_RESTORE_DIR}"
                        elif [[ "\$FILE" == *.tar ]]; then
                            tar -xf "\$FILE" -C "${TEMP_RESTORE_DIR}"
                        elif [[ "\$FILE" == *.zip ]]; then
                            unzip "\$FILE" -d "${TEMP_RESTORE_DIR}"
                        else
                            echo "No extraction needed."
                        fi
                    """
                }
            }
        }

        stage('PostgreSQL Restore') {
            steps {
                script {
                    echo "Searching for SQL file..."

                    env.SQL_FILE = sh(
                        script: "find ${TEMP_RESTORE_DIR} -name '*.sql' | head -n 1",
                        returnStdout: true
                    ).trim()

                    if (!env.SQL_FILE) {
                        error("No .sql file found to restore!")
                    }

                    echo "Found SQL file: ${env.SQL_FILE}"

                    sh """
                        export PGPASSWORD="${PGPASSWORD}"

                        psql -h ${PGHOST} -U ${PGUSER} -d ${PGDATABASE} -f "${env.SQL_FILE}"
                    """

                    echo "✓ PostgreSQL Restore Completed Successfully!"
                }
            }
        }
    }

    post {
        always {
            sh "rm -rf ${TEMP_RESTORE_DIR}"
        }
    }
}
