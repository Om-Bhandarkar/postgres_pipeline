pipeline {
    agent any

    parameters {
        file(
            name: 'UPLOADED_BACKUP_FILE',
            description: 'Upload your PostgreSQL .dump file to restore'
        )
    }

    environment {
        RESTORE_LOCATION = "${WORKSPACE}/restored_files"
    }

    stages {

        stage('Initialize') {
            steps {
                script {
                    echo "Initializing PostgreSQL Restore Pipeline..."
                    echo "Workspace: ${WORKSPACE}"

                    sh "mkdir -p '${RESTORE_LOCATION}'"

                    // FILE PARAM FIX — Correct full path
                    if (!params.UPLOADED_BACKUP_FILE) {
                        error "❌ No backup file uploaded. Upload a .dump file and try again."
                    }

                    // FULL PATH OF THE UPLOADED FILE
                    env.UPLOADED_FILE = "${WORKSPACE}/${params.UPLOADED_BACKUP_FILE}"

                    echo "Uploaded file name: ${params.UPLOADED_BACKUP_FILE}"
                    echo "Full file path: ${env.UPLOADED_FILE}"

                    // timestamp
                    env.TIMESTAMP = sh(script: 'date +%Y%m%d_%H%M%S', returnStdout: true).trim()
                }
            }
        }

        stage('Restore Database') {
            steps {
                script {
                    if (!env.UPLOADED_FILE.endsWith(".dump")) {
                        error "❌ Only .dump PostgreSQL backup files are supported."
                    }

                    sh """
                        echo "Checking if uploaded .dump file exists..."
                        ls -lh "${env.UPLOADED_FILE}" || (echo "File not found!" && exit 1)

                        echo "Copying dump file to restore folder..."
                        cp "${env.UPLOADED_FILE}" "${RESTORE_LOCATION}/restore.dump"

                        echo "Ensuring restore_db exists..."
                        PGPASSWORD='mypassword' createdb -U postgres restore_db || echo "DB already exists"

                        echo "Running PostgreSQL restore..."
                        PGPASSWORD='mypassword' pg_restore \
                            --no-owner \
                            --role=postgres \
                            -U postgres \
                            -d restore_db \
                            "${RESTORE_LOCATION}/restore.dump"

                        echo "✓ Restore Completed Successfully"
                    """
                }
            }
        }

        stage('Summary') {
            steps {
                script {
                    echo "===== RESTORE SUMMARY ====="
                    sh "ls -lh '${RESTORE_LOCATION}'"
                }
            }
        }
    }

    post {
        success {
            echo "🎉 PostgreSQL Restore Successfully Completed!"
        }
        failure {
            echo "❌ Restore failed — check errors above."
        }
        always {
            echo "Pipeline execution complete."
        }
    }
}
