pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
metadata:
  labels:
    jenkins: agent
    app: pyexample3-ci
spec:
  containers:
  - name: python
    image: python:3.13
    command:
    - cat
    tty: true
    resources:
      requests:
        memory: "768Mi"
        cpu: "500m"
      limits:
        memory: "1.5Gi"
        cpu: "1000m"
'''
            defaultContainer 'python'
        }
    }

    environment {
        POETRY_VERSION = '1.7.1'
        POETRY_HOME = '/opt/poetry'
        POETRY_VIRTUALENVS_IN_PROJECT = 'true'
        POETRY_NO_INTERACTION = '1'
        PYTHONUNBUFFERED = '1'
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '30', artifactNumToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    // Fix git safe directory issue in containers
                    sh 'git config --global --add safe.directory "*"'

                    env.GIT_COMMIT_SHORT = sh(
                        script: "git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()
                    env.GIT_BRANCH_NAME = sh(
                        script: "git rev-parse --abbrev-ref HEAD",
                        returnStdout: true
                    ).trim()
                    echo "Building commit ${env.GIT_COMMIT_SHORT} on branch ${env.GIT_BRANCH_NAME}"
                }
            }
        }

        stage('Test') {
            options {
                timeout(time: 10, unit: 'MINUTES')
            }
            steps {
                container('python') {
                    sh '''
                        echo "=== Installing Poetry ==="
                        pip install --no-cache-dir poetry==${POETRY_VERSION}

                        echo "=== Poetry version ==="
                        poetry --version

                        echo "=== Installing dependencies ==="
                        poetry install --with dev --no-root

                        echo "=== Running tests with coverage ==="
                        poetry run pytest \
                            --junitxml=test-results.xml \
                            --cov=app \
                            --cov-report=markdown:coverage.md \
                            --cov-report=term \
                            --cov-report=html:coverage-html \
                            -v

                        echo "=== Coverage Summary ==="
                        cat coverage.md
                    '''
                }
            }
        }

        stage('Snyk SAST') {
            options {
                timeout(time: 10, unit: 'MINUTES')
            }
            steps {
                container('python') {
                    catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                        withCredentials([
                            string(credentialsId: 'snyk-api-secret', variable: 'SNYK_TOKEN'),
                            string(credentialsId: 'snyk-org', variable: 'SNYK_ORG')
                        ]) {
                            sh '''
                                echo "=== Installing Snyk CLI ==="
                                curl -Lo /usr/local/bin/snyk https://static.snyk.io/cli/latest/snyk-linux
                                chmod +x /usr/local/bin/snyk

                                echo "=== Authenticating with Snyk ==="
                                snyk auth ${SNYK_TOKEN}

                                echo "=== Running Snyk Code (SAST) scan ==="
                                snyk code test \
                                    --org=${SNYK_ORG} \
                                    --severity-threshold=medium \
                                    --remote-repo-url=https://github.com/ldorg/pyexample3 \
                                    --json-file-output=snyk-sast-results.json \
                                    --sarif-file-output=snyk-sast-results.sarif \
                                    || echo "Snyk SAST found issues (exit code: $?)"

                                echo "=== SAST Scan Summary ==="
                                if [ -f snyk-sast-results.json ]; then
                                    python -c "import json; data=json.load(open('snyk-sast-results.json')); print('SAST scan completed')" || true
                                fi
                                
                                echo "=== Checking for output files ==="
                                ls -lah snyk-sast-results.* 2>/dev/null || echo "No SAST result files found"
                                echo "Working directory: $(pwd)"
                            '''
                        }
                    }
                }
            }
        }

        stage('Snyk SCA') {
            options {
                timeout(time: 10, unit: 'MINUTES')
            }
            steps {
                container('python') {
                    catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                        withCredentials([
                            string(credentialsId: 'snyk-api-secret', variable: 'SNYK_TOKEN'),
                            string(credentialsId: 'snyk-org', variable: 'SNYK_ORG')
                        ]) {
                            sh '''
                                echo "=== Re-authenticating with Snyk ==="
                                snyk auth ${SNYK_TOKEN}

                                echo "=== Installing project dependencies for SCA scan ==="
                                # Dependencies should already be installed from test stage
                                poetry install --no-dev || poetry install --without dev

                                echo "=== Running Snyk Open Source (SCA) scan ==="
                                snyk test \
                                    --org=${SNYK_ORG} \
                                    --severity-threshold=medium \
                                    --remote-repo-url=https://github.com/ldorg/pyexample3 \
                                    --json-file-output=snyk-sca-results.json \
                                    --sarif-file-output=snyk-sca-results.sarif \
                                    || echo "Snyk SCA found vulnerabilities (exit code: $?)"

                                echo "=== SCA Scan Summary ==="
                                if [ -f snyk-sca-results.json ]; then
                                    python -c "import json; data=json.load(open('snyk-sca-results.json')); print('SCA scan completed')" || true
                                fi
                                
                                echo "=== Checking for output files ==="
                                ls -lah snyk-sca-results.* 2>/dev/null || echo "No SCA result files found"
                                echo "Working directory: $(pwd)"
                            '''
                        }
                    }
                }
            }
        }
    }
    stage('Security Scan') {
            steps {
                registerSecurityScan(
                    // Security Scan to include
                    artifacts: "snyk-sca-results.sarif",
                    format: "sarif",
                    archive: true
                )
            }
        }

    post {
        always {
            // Publish JUnit test results (automatically registers with CloudBees Unify)
            junit testResults: 'test-results.xml', allowEmptyResults: false

            // Archive coverage reports
            // archiveArtifacts artifacts: 'coverage.md,coverage-html/**', allowEmptyArchive: false, fingerprint: true

            // Archive Snyk scan results
            archiveArtifacts artifacts: '**/*.sarif,**/*.json', allowEmptyArchive: true, fingerprint: true

            // Register security scans with CloudBees Unify
            // script {
            //     if (fileExists('snyk-sast-results.sarif')) {
            //         registerSecurityScan(
            //             artifacts: 'snyk-sast-results.sarif',
            //             format: 'sarif',
            //             archive: true
            //         )
            //         echo "✓ Registered SAST results with CloudBees Unify"
            //     } else {
            //         echo "⚠ snyk-sast-results.sarif not found, skipping registration"
            //     }
                
            //     if (fileExists('snyk-sca-results.sarif')) {
            //         registerSecurityScan(
            //             artifacts: 'snyk-sca-results.sarif',
            //             format: 'sarif',
            //             archive: true
            //         )
            //         echo "✓ Registered SCA results with CloudBees Unify"
            //     } else {
            //         echo "⚠ snyk-sca-results.sarif not found, skipping registration"
            //     }
            // }
        }

        success {
            echo "✓ Pipeline completed successfully!"
            echo "  Commit: ${env.GIT_COMMIT_SHORT}"
            echo "  Branch: ${env.GIT_BRANCH_NAME}"
        }

        unstable {
            echo "⚠ Pipeline completed with warnings"
            echo "  This may indicate security findings or test failures that didn't fail the build"
        }

        failure {
            echo "✗ Pipeline failed!"
            echo "  Check the logs above for error details"
        }
    }
}
