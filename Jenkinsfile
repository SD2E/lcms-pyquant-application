#!groovy

pipeline {
    agent any
    environment {
        AGAVE_GET_DATA    = 1
        AGAVE_JOB_TIMEOUT = 1800
        AGAVE_JOB_GET_DIR = "job_output"
        AGAVE_DATA_URI    = "agave://data-tacc-work-cylu/sd2e-data/lcms-pyquant-application/test_data"
        CONTAINER_REPO    = "lcms-pyquant"
        CONTAINER_TAG     = "test"
        AGAVE_CACHE_DIR   = "${HOME}/credentials_cache/${JOB_BASE_NAME}"
        AGAVE_JSON_PARSER = "jq"
        AGAVE_TENANTID    = "sd2e"
        AGAVE_APISERVER   = "https://api.sd2e.org"
        AGAVE_USERNAME    = "sd2etest"
        AGAVE_PASSWORD    = credentials('sd2etest-tacc-password')
        REGISTRY_USERNAME = "sd2etest"
        REGISTRY_PASSWORD = credentials('sd2etest-dockerhub-password')
        REGISTRY_ORG      = credentials('sd2etest-dockerhub-org')
        PATH = "${HOME}/bin:${HOME}/sd2e-cloud-cli/bin:${env.PATH}"
        }
    stages {

        stage('Create Oauth client') { 
            steps {
                sh "make-session-client ${JOB_BASE_NAME} ${JOB_BASE_NAME}-${BUILD_ID}"
            }
        }
        stage('Clean up previous test data') {
            when { 
                environment name: 'AGAVE_GET_DATA', value: '1' 
            }
            steps {
                echo "Removing test_data"
                sh "rm -rf test_data || true "
            }
        }
        stage('(Copy in test data)') {
            when { not {
                    expression { fileExists('test_data') }
                   }
            }
            steps {
                sh "files-get -r -S data-tacc-work-cylu sd2e-data/lcms-pyquant-application/test_data > files-get.log 2>&1"
                sh "ls -alth test_data"
            }
        }
        stage('Build app container') { 
            steps {
                sh "apps-build-container -O ${REGISTRY_USERNAME} --image ${CONTAINER_REPO} --tag ${CONTAINER_TAG}"
            }
        }
        stage('Run functional test(s)') { 
            steps {
                sh "tests/run_functional_test.sh ${REGISTRY_USERNAME}/${CONTAINER_REPO}:${CONTAINER_TAG} test_data"
            }
        }
        stage('Deploy to TACC.cloud') { 
            steps {
                sh "apps-deploy -T -O ${REGISTRY_USERNAME} --image ${CONTAINER_REPO} --tag ${CONTAINER_TAG}"
                sh "cat deploy-*"
            }
        }
        stage('Run a test job') { 
            steps {
                // Run job
                sh "run-test-job deploy-${AGAVE_USERNAME}-job.json ${AGAVE_JOB_TIMEOUT}"
                // Get outputs
                sh "get-test-job-outputs deploy-${AGAVE_USERNAME}-job.json.jobid"
                // Evaluate results
            }
        }
        stage('Validate results') {
            steps {
                sh "python -m pytest tests/validate_job --job-directory ${AGAVE_JOB_GET_DIR}"
            }
        }

    }
    post {
        always {
            sh "delete-session-client ${JOB_BASE_NAME} ${JOB_BASE_NAME}-${BUILD_ID}"
        }
        success {
            deleteDir()
        }
    }
}
