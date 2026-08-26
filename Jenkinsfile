pipeline {

    agent any

    environment {
        MVOLA_APIM_URL = 'https://13.49.18.52:9443'
    }

    stages {

        stage('Checkout Governance') {
            steps {
                checkout scm

                sh '''
                    set -e

                    echo "======================================"
                    echo "AXIAN CENTRAL GOVERNANCE"
                    echo "======================================"

                    echo "Governance Version:"
                    cat version

                    echo ""
                    echo "Ruleset:"
                    ls -l rules/

                    echo ""
                    echo "Policy:"
                    ls -l policy/
                '''
            }
        }


        stage('Validate Governance Package') {
            steps {
                sh '''
                    set -e

                    test -f version
                    test -f rules/group-api-standards.yaml
                    test -f policy/group-api-governance.json

                    echo "Governance package validation PASSED."
                '''
            }
        }


        stage('Test Mvola APIM Connection') {
            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'mvola-governance-admin',
                        usernameVariable: 'MVOLA_USER',
                        passwordVariable: 'MVOLA_PASS'
                    )
                ]) {

                    sh '''
                        set -e

                        echo "Testing Mvola APIM connection..."

                        HTTP_CODE=$(curl -sk \
                            -o /tmp/mvola-response.json \
                            -w "%{http_code}" \
                            -u "$MVOLA_USER:$MVOLA_PASS" \
                            -H "Accept: application/json" \
                            "$MVOLA_APIM_URL/api/am/governance/v1/rulesets")

                        echo "HTTP Status: $HTTP_CODE"

                        if [ "$HTTP_CODE" != "200" ]; then
                            echo "ERROR: Unable to connect to Mvola Governance API."
                            cat /tmp/mvola-response.json
                            exit 1
                        fi

                        echo "Mvola APIM connection successful."
                    '''
                }
            }
        }


        stage('Find Mvola Ruleset') {
            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'mvola-governance-admin',
                        usernameVariable: 'MVOLA_USER',
                        passwordVariable: 'MVOLA_PASS'
                    )
                ]) {

                    sh '''
                        set -e

                        echo "Searching for Group Level API Standards..."

                        curl -sk \
                            -u "$MVOLA_USER:$MVOLA_PASS" \
                            -H "Accept: application/json" \
                            --get \
                            --data-urlencode "query=name:Group Level API Standards" \
                            "$MVOLA_APIM_URL/api/am/governance/v1/rulesets" \
                            > ruleset-response.json

                        echo "Ruleset response:"
                        jq . ruleset-response.json

                        RULESET_ID=$(jq -r '.list[0].id // empty' ruleset-response.json)

                        if [ -z "$RULESET_ID" ]; then

                            echo "Ruleset does not exist."
                            echo "CREATE" > ruleset-action

                        else

                            echo "Ruleset found."
                            echo "Ruleset ID: $RULESET_ID"

                            echo "$RULESET_ID" > ruleset-id
                            echo "UPDATE" > ruleset-action

                        fi
                    '''
                }
            }
        }


        stage('Create Mvola Ruleset') {

            when {
                expression {
                    fileExists('ruleset-action') &&
                    readFile('ruleset-action').trim() == 'CREATE'
                }
            }

            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'mvola-governance-admin',
                        usernameVariable: 'MVOLA_USER',
                        passwordVariable: 'MVOLA_PASS'
                    )
                ]) {

                    sh '''
                        set -e

                        echo "Creating governance ruleset..."

                        curl -sk \
                            -u "$MVOLA_USER:$MVOLA_PASS" \
                            -X POST \
                            "$MVOLA_APIM_URL/api/am/governance/v1/rulesets" \
                            -H "Accept: application/json" \
                            -F "name=Group Level API Standards" \
                            -F "description=Axian Group API Governance Standards" \
                            -F "ruleType=API_METADATA" \
                            -F "artifactType=REST_API" \
                            -F "ruleCategory=SPECTRAL" \
                            -F "rulesetContent=<rules/group-api-standards.yaml" \
                            > ruleset-created.json

                        echo "Create response:"
                        jq . ruleset-created.json

                        RULESET_ID=$(jq -r '.id // empty' ruleset-created.json)

                        if [ -z "$RULESET_ID" ]; then
                            echo "ERROR: Ruleset creation failed."
                            exit 1
                        fi

                        echo "$RULESET_ID" > ruleset-id

                        echo "Ruleset created successfully."
                        echo "Ruleset ID: $RULESET_ID"
                    '''
                }
            }
        }


        stage('Update Mvola Ruleset') {

            when {
                expression {
                    fileExists('ruleset-action') &&
                    readFile('ruleset-action').trim() == 'UPDATE'
                }
            }

            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'mvola-governance-admin',
                        usernameVariable: 'MVOLA_USER',
                        passwordVariable: 'MVOLA_PASS'
                    )
                ]) {

                    sh '''
                        set -e

                        RULESET_ID=$(cat ruleset-id)

                        echo "Updating ruleset:"
                        echo "$RULESET_ID"

                        curl -sk \
                            -u "$MVOLA_USER:$MVOLA_PASS" \
                            -X PUT \
                            "$MVOLA_APIM_URL/api/am/governance/v1/rulesets/$RULESET_ID" \
                            -H "Accept: application/json" \
                            -F "name=Group Level API Standards" \
                            -F "description=Axian Group API Governance Standards" \
                            -F "ruleType=API_METADATA" \
                            -F "artifactType=REST_API" \
                            -F "ruleCategory=SPECTRAL" \
                            -F "rulesetContent=<rules/group-api-standards.yaml" \
                            > ruleset-updated.json

                        echo "Update response:"
                        jq . ruleset-updated.json

                        echo "Ruleset updated successfully."
                    '''
                }
            }
        }


        stage('Verify Mvola Ruleset') {

            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'mvola-governance-admin',
                        usernameVariable: 'MVOLA_USER',
                        passwordVariable: 'MVOLA_PASS'
                    )
                ]) {

                    sh '''
                        set -e

                        RULESET_ID=$(cat ruleset-id)

                        echo "Verifying ruleset: $RULESET_ID"

                        curl -sk \
                            -u "$MVOLA_USER:$MVOLA_PASS" \
                            -H "Accept: application/json" \
                            "$MVOLA_APIM_URL/api/am/governance/v1/rulesets/$RULESET_ID" \
                            > ruleset-verify.json

                        jq . ruleset-verify.json

                        NAME=$(jq -r '.name // empty' ruleset-verify.json)

                        if [ "$NAME" != "Group Level API Standards" ]; then
                            echo "ERROR: Ruleset verification failed."
                            exit 1
                        fi

                        echo "Ruleset verification PASSED."
                    '''
                }
            }
        }


        stage('Find Mvola Governance Policy') {

            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'mvola-governance-admin',
                        usernameVariable: 'MVOLA_USER',
                        passwordVariable: 'MVOLA_PASS'
                    )
                ]) {

                    sh '''
                        set -e

                        echo "Searching for Group API Design Standards policy..."

                        curl -sk \
                            -u "$MVOLA_USER:$MVOLA_PASS" \
                            -H "Accept: application/json" \
                            --get \
                            --data-urlencode "query=name:Group API Design Standards" \
                            "$MVOLA_APIM_URL/api/am/governance/v1/policies" \
                            > policy-response.json

                        echo "Policy response:"
                        jq . policy-response.json

                        POLICY_ID=$(jq -r '.list[0].id // empty' policy-response.json)

                        if [ -z "$POLICY_ID" ]; then
                            echo "ERROR: Group API Design Standards policy not found."
                            exit 1
                        fi

                        echo "$POLICY_ID" > policy-id

                        echo "Policy found:"
                        echo "$POLICY_ID"
                    '''
                }
            }
        }


        stage('Get Existing Mvola Policy') {

            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'mvola-governance-admin',
                        usernameVariable: 'MVOLA_USER',
                        passwordVariable: 'MVOLA_PASS'
                    )
                ]) {

                    sh '''
                        set -e

                        POLICY_ID=$(cat policy-id)

                        echo "Getting policy: $POLICY_ID"

                        curl -sk \
                            -u "$MVOLA_USER:$MVOLA_PASS" \
                            -H "Accept: application/json" \
                            "$MVOLA_APIM_URL/api/am/governance/v1/policies/$POLICY_ID" \
                            > policy-current.json

                        jq . policy-current.json
                    '''
                }
            }
        }


        stage('Update Mvola Policy') {

            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'mvola-governance-admin',
                        usernameVariable: 'MVOLA_USER',
                        passwordVariable: 'MVOLA_PASS'
                    )
                ]) {

                    sh '''
                        set -e

                        POLICY_ID=$(cat policy-id)
                        RULESET_ID=$(cat ruleset-id)

                        echo "Updating policy..."
                        echo "Policy ID : $POLICY_ID"
                        echo "Ruleset ID: $RULESET_ID"

                        jq \
                            --arg ruleset "$RULESET_ID" \
                            '.rulesets = ((.rulesets // []) + [$ruleset] | unique)' \
                            policy-current.json \
                            > policy-updated.json

                        echo "Updated policy:"
                        jq . policy-updated.json

                        curl -sk \
                            -u "$MVOLA_USER:$MVOLA_PASS" \
                            -X PUT \
                            "$MVOLA_APIM_URL/api/am/governance/v1/policies/$POLICY_ID" \
                            -H "Accept: application/json" \
                            -H "Content-Type: application/json" \
                            --data @policy-updated.json \
                            > policy-updated-response.json

                        echo "Policy update response:"
                        jq . policy-updated-response.json

                        echo "Policy updated successfully."
                    '''
                }
            }
        }


        stage('Verify Mvola Policy') {

            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'mvola-governance-admin',
                        usernameVariable: 'MVOLA_USER',
                        passwordVariable: 'MVOLA_PASS'
                    )
                ]) {

                    sh '''
                        set -e

                        POLICY_ID=$(cat policy-id)

                        echo "Verifying policy..."

                        curl -sk \
                            -u "$MVOLA_USER:$MVOLA_PASS" \
                            -H "Accept: application/json" \
                            "$MVOLA_APIM_URL/api/am/governance/v1/policies/$POLICY_ID" \
                            > policy-verify.json

                        jq . policy-verify.json

                        echo "Policy verification completed."
                    '''
                }
            }
        }
    }


    post {

        success {
            echo '''
========================================
AXIAN GOVERNANCE RELEASE SUCCESS
========================================
Mvola governance ruleset and policy
have been successfully synchronized.
========================================
'''
        }

        failure {
            echo '''
========================================
AXIAN GOVERNANCE RELEASE FAILED
========================================
Mvola governance synchronization failed.
========================================
'''
        }
    }
}
