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

        /* ---------------------- INITIALIZE ---------------------- */
        stage('Initialize') {
            steps {
                script {

                    echo "Initializing PostgreSQL Restore Pipeline..."
                    echo "Workspace: ${WORKSPACE}"

                    // Create temp restore directory
                    sh """
                        mkdir -p "${RESTORE_LOCATION}"
                    """

                    // Validate upload
                    if (!params.UPLOADED_BACKUP_FILE) {
                        error "❌ No backup file uploaded. Upload a .dump file and try again."
                    }

                    env.UPLOADED_FILE = params.UPLOADED_BACKUP_FILE

                    echo "Uploaded File: ${env.UPLOADED_FILE}"

                    // timestamp
                    env.TIMESTAMP = sh(
                        script: 'date +%Y%m%d_%H%M%S',
                        returnStdout: true
                    ).trim()
                }
            }
        }

        /* ---------------------- RESTORE PROCESS ---------------------- */
        stage('Restore Database') {
            steps {
                script {

                    if (!env.UPLOADED_FILE.endsWith(".dump")) {
                        error "❌ Only .dump PostgreSQL backup files are supported."
                    }

                    echo "✓ Valid .dump file detected"
                    echo "Starting PostgreSQL Restore Process..."

                    sh """
                        echo "Copying dump file to Jenkins temp restore folder..."
                        cp "${env.UPLOADED_FILE}" "${RESTORE_LOCATION}/restore.dump"

                        echo "Ensuring target PostgreSQL database exists..."
                        PGPASSWORD='mypassword' createdb -U postgres restore_db || echo "Database already exists"

                        echo "Running pg_restore..."
                        PGPASSWORD='mypassword' pg_restore \
                            --no-owner \
                            --role=postgres \
                            -U postgres \
                            -d restore_db \
                            "${RESTORE_LOCATION}/restore.dump"

                        echo "✓ PostgreSQL Restore Completed Successfully"
                    """

                    currentBuild.description = "Restore done → DB: restore_db | File: restore.dump"
                }
            }
        }

        /* ---------------------- SUMMARY ---------------------- */
        stage('Summary') {
            steps {
                script {

                    echo "============== RESTORE SUMMARY =============="
                    echo "Temporary restore folder: ${RESTORE_LOCATION}"

                    sh """
                        echo "Restored dump file:"
                        ls -lh "${RESTORE_LOCATION}"
                    """

                    echo "Database restored to: restore_db"
                    echo "============================================="
                }
            }
        }
    }

    /* ---------------------- POST EXECUTION ---------------------- */
    post {
        success {
            echo "🎉 PostgreSQL Restore Pipeline Completed Successfully!"
        }
        failure {
            echo "❌ Restore failed — check Jenkins log above."
        }
        always {
            echo "Pipeline execution complete."
        }
    }
}
