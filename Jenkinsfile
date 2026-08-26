pipeline {

    agent any

    environment {

        // ==========================================
        // APIM ENVIRONMENTS
        // ==========================================

        MVOLA_APIM_URL = 'https://13.49.18.52:9443'
        GROUP_APIM_URL = 'https://13.60.222.145:9443'
        TIGO_APIM_URL  = 'https://13.62.128.136:9443'
    }

    stages {

        // =========================================================
        // CHECKOUT
        // =========================================================

        stage('Checkout Governance') {

            steps {

                checkout scm

                sh '''
                    set -e

                    echo "======================================"
                    echo "AXIAN CENTRAL API GOVERNANCE"
                    echo "======================================"

                    echo ""
                    echo "Governance Version:"
                    cat version

                    echo ""
                    echo "Ruleset:"
                    ls -lh rules/

                    echo ""
                    echo "Policy:"
                    ls -lh policy/
                '''
            }
        }


        // =========================================================
        // VALIDATE CENTRAL GOVERNANCE PACKAGE
        // =========================================================

        stage('Validate Governance Package') {

            steps {

                sh '''
                    set -e

                    echo "Validating governance package..."

                    test -f version
                    test -f rules/group-api-standards.yaml

                    echo ""
                    echo "Governance package validation PASSED."
                '''
            }
        }


        // =========================================================
        // SYNC GOVERNANCE TO ALL ENVIRONMENTS
        // =========================================================

        stage('Deploy Governance to All Environments') {

            steps {

                script {

                    def environments = [

                        [
                            name: 'GROUP',
                            url: env.GROUP_APIM_URL,
                            credentialId: 'group-governance-admin',
                            rulesetName: 'Group Level API Standards',
                            policyName: 'Group API Design Standards'
                        ],

                        [
                            name: 'MVOLA',
                            url: env.MVOLA_APIM_URL,
                            credentialId: 'mvola-governance-admin',
                            rulesetName: 'Group Level API Standards',
                            policyName: 'Group API Design Standards'
                        ],

                        [
                            name: 'TIGO',
                            url: env.TIGO_APIM_URL,
                            credentialId: 'tigo-governance-admin',
                            rulesetName: 'Group Level API Standards',
                            policyName: 'Group API Design Standards'
                        ]
                    ]


                    for (target in environments) {

                        echo ""
                        echo "=============================================="
                        echo " GOVERNANCE DEPLOYMENT: ${target.name}"
                        echo "=============================================="
                        echo "APIM URL: ${target.url}"
                        echo "Ruleset : ${target.rulesetName}"
                        echo "Policy  : ${target.policyName}"
                        echo "=============================================="


                        withCredentials([
                            usernamePassword(
                                credentialsId: target.credentialId,
                                usernameVariable: 'APIM_USER',
                                passwordVariable: 'APIM_PASS'
                            )
                        ]) {

                            // =================================================
                            // TEST CONNECTION
                            // =================================================

                            sh """
                                set -e

                                echo ""
                                echo "Testing ${target.name} APIM connection..."

                                HTTP_CODE=\\$(curl -sk \\
                                    -o /tmp/${target.name}-connection.json \\
                                    -w "%{http_code}" \\
                                    -u "\\$APIM_USER:\\$APIM_PASS" \\
                                    -H "Accept: application/json" \\
                                    "${target.url}/api/am/governance/v1/rulesets")

                                echo "HTTP Status: \\$HTTP_CODE"

                                if [ "\\$HTTP_CODE" != "200" ]; then
                                    echo ""
                                    echo "ERROR: Unable to connect to ${target.name}."
                                    cat /tmp/${target.name}-connection.json
                                    exit 1
                                fi

                                echo "${target.name} connection successful."
                            """


                            // =================================================
                            // FIND RULESET
                            // =================================================

                            sh """
                                set -e

                                echo ""
                                echo "Finding ruleset in ${target.name}..."

                                curl -sk \\
                                    -u "\\$APIM_USER:\\$APIM_PASS" \\
                                    -H "Accept: application/json" \\
                                    "${target.url}/api/am/governance/v1/rulesets?limit=100" \\
                                    > ${target.name}-rulesets.json

                                echo ""
                                echo "Available rulesets:"

                                jq '.list[] | {
                                    id,
                                    name,
                                    ruleType,
                                    artifactType
                                }' ${target.name}-rulesets.json

                                RULESET_ID=\\$(jq -r '
                                    .list[]
                                    | select(.name == "${target.rulesetName}")
                                    | .id
                                ' ${target.name}-rulesets.json | head -n 1)

                                if [ -z "\\$RULESET_ID" ]; then

                                    echo ""
                                    echo "Ruleset does not exist in ${target.name}."
                                    echo "CREATE" > ${target.name}-ruleset-action

                                else

                                    echo ""
                                    echo "Ruleset found:"
                                    echo "\\$RULESET_ID"

                                    echo "\\$RULESET_ID" > ${target.name}-ruleset-id
                                    echo "UPDATE" > ${target.name}-ruleset-action

                                fi
                            """


                            // =================================================
                            // CREATE RULESET
                            // =================================================

                            sh """
                                set -e

                                ACTION=\\$(cat ${target.name}-ruleset-action)

                                if [ "\\$ACTION" != "CREATE" ]; then
                                    echo "Ruleset already exists."
                                    echo "Skipping CREATE."
                                    exit 0
                                fi

                                echo ""
                                echo "Creating ruleset in ${target.name}..."

                                HTTP_CODE=\\$(curl -sk \\
                                    -u "\\$APIM_USER:\\$APIM_PASS" \\
                                    -o ${target.name}-ruleset-created.json \\
                                    -w "%{http_code}" \\
                                    -X POST \\
                                    "${target.url}/api/am/governance/v1/rulesets" \\
                                    -H "Accept: application/json" \\
                                    -F "name=${target.rulesetName}" \\
                                    -F "description=Axian Group API Governance Standards" \\
                                    -F "ruleType=API_METADATA" \\
                                    -F "artifactType=REST_API" \\
                                    -F "ruleCategory=SPECTRAL" \\
                                    -F "rulesetContent=@rules/group-api-standards.yaml"
                                )

                                echo ""
                                echo "HTTP Status: \\$HTTP_CODE"

                                jq . ${target.name}-ruleset-created.json || true

                                if [ "\\$HTTP_CODE" -lt 200 ] || [ "\\$HTTP_CODE" -ge 300 ]; then
                                    echo ""
                                    echo "ERROR: Ruleset creation failed in ${target.name}."
                                    exit 1
                                fi

                                RULESET_ID=\\$(jq -r '.id // empty' ${target.name}-ruleset-created.json)

                                if [ -z "\\$RULESET_ID" ]; then
                                    echo ""
                                    echo "ERROR: Ruleset ID was not returned."
                                    exit 1
                                fi

                                echo "\\$RULESET_ID" > ${target.name}-ruleset-id

                                echo ""
                                echo "Ruleset created successfully."
                                echo "Ruleset ID: \\$RULESET_ID"
                            """


                            // =================================================
                            // UPDATE RULESET
                            // =================================================

                            sh """
                                set -e

                                ACTION=\\$(cat ${target.name}-ruleset-action)

                                if [ "\\$ACTION" != "UPDATE" ]; then
                                    echo "Ruleset was created."
                                    echo "Skipping UPDATE."
                                    exit 0
                                fi

                                RULESET_ID=\\$(cat ${target.name}-ruleset-id)

                                echo ""
                                echo "Updating ruleset in ${target.name}..."
                                echo "Ruleset ID: \\$RULESET_ID"

                                HTTP_CODE=\\$(curl -sk \\
                                    -u "\\$APIM_USER:\\$APIM_PASS" \\
                                    -o ${target.name}-ruleset-updated.json \\
                                    -w "%{http_code}" \\
                                    -X PUT \\
                                    "${target.url}/api/am/governance/v1/rulesets/\\$RULESET_ID" \\
                                    -H "Accept: application/json" \\
                                    -F "name=${target.rulesetName}" \\
                                    -F "description=Axian Group API Governance Standards" \\
                                    -F "ruleType=API_METADATA" \\
                                    -F "artifactType=REST_API" \\
                                    -F "ruleCategory=SPECTRAL" \\
                                    -F "rulesetContent=@rules/group-api-standards.yaml"
                                )

                                echo ""
                                echo "HTTP Status: \\$HTTP_CODE"

                                jq . ${target.name}-ruleset-updated.json || true

                                if [ "\\$HTTP_CODE" -lt 200 ] || [ "\\$HTTP_CODE" -ge 300 ]; then
                                    echo ""
                                    echo "ERROR: Ruleset update failed in ${target.name}."
                                    exit 1
                                fi

                                echo ""
                                echo "${target.name} ruleset updated successfully."
                            """


                            // =================================================
                            // VERIFY RULESET
                            // =================================================

                            sh """
                                set -e

                                RULESET_ID=\\$(cat ${target.name}-ruleset-id)

                                echo ""
                                echo "Verifying ${target.name} ruleset..."

                                curl -sk \\
                                    -u "\\$APIM_USER:\\$APIM_PASS" \\
                                    -H "Accept: application/json" \\
                                    "${target.url}/api/am/governance/v1/rulesets/\\$RULESET_ID" \\
                                    > ${target.name}-ruleset-verify.json

                                jq . ${target.name}-ruleset-verify.json

                                NAME=\\$(jq -r '.name // empty' ${target.name}-ruleset-verify.json)

                                if [ "\\$NAME" != "${target.rulesetName}" ]; then
                                    echo ""
                                    echo "ERROR: Ruleset verification failed in ${target.name}."
                                    exit 1
                                fi

                                echo ""
                                echo "${target.name} ruleset verification PASSED."
                            """


                            // =================================================
                            // FIND POLICY
                            // =================================================

                            sh """
                                set -e

                                echo ""
                                echo "Finding governance policy in ${target.name}..."

                                curl -sk \\
                                    -u "\\$APIM_USER:\\$APIM_PASS" \\
                                    -H "Accept: application/json" \\
                                    --get \\
                                    --data-urlencode "query=name:${target.policyName}" \\
                                    "${target.url}/api/am/governance/v1/policies" \\
                                    > ${target.name}-policy-response.json

                                jq . ${target.name}-policy-response.json

                                POLICY_ID=\\$(jq -r '.list[0].id // empty' ${target.name}-policy-response.json)

                                if [ -z "\\$POLICY_ID" ]; then
                                    echo ""
                                    echo "ERROR: Policy '${target.policyName}' not found in ${target.name}."
                                    exit 1
                                fi

                                echo "\\$POLICY_ID" > ${target.name}-policy-id

                                echo ""
                                echo "Policy found:"
                                echo "\\$POLICY_ID"
                            """


                            // =================================================
                            // GET POLICY
                            // =================================================

                            sh """
                                set -e

                                POLICY_ID=\\$(cat ${target.name}-policy-id)

                                echo ""
                                echo "Getting existing policy..."

                                curl -sk \\
                                    -u "\\$APIM_USER:\\$APIM_PASS" \\
                                    -H "Accept: application/json" \\
                                    "${target.url}/api/am/governance/v1/policies/\\$POLICY_ID" \\
                                    > ${target.name}-policy-current.json

                                jq . ${target.name}-policy-current.json
                            """


                            // =================================================
                            // UPDATE POLICY
                            // =================================================

                            sh """
                                set -e

                                POLICY_ID=\\$(cat ${target.name}-policy-id)
                                RULESET_ID=\\$(cat ${target.name}-ruleset-id)

                                echo ""
                                echo "Associating ruleset with policy..."

                                jq \\
                                    --arg ruleset "\\$RULESET_ID" \\
                                    '.rulesets = ((.rulesets // []) + [\\$ruleset] | unique)' \\
                                    ${target.name}-policy-current.json \\
                                    > ${target.name}-policy-updated.json

                                echo ""
                                echo "Updated policy:"
                                jq . ${target.name}-policy-updated.json

                                HTTP_CODE=\\$(curl -sk \\
                                    -u "\\$APIM_USER:\\$APIM_PASS" \\
                                    -o ${target.name}-policy-updated-response.json \\
                                    -w "%{http_code}" \\
                                    -X PUT \\
                                    "${target.url}/api/am/governance/v1/policies/\\$POLICY_ID" \\
                                    -H "Accept: application/json" \\
                                    -H "Content-Type: application/json" \\
                                    --data @${target.name}-policy-updated.json
                                )

                                echo ""
                                echo "HTTP Status: \\$HTTP_CODE"

                                jq . ${target.name}-policy-updated-response.json || true

                                if [ "\\$HTTP_CODE" -lt 200 ] || [ "\\$HTTP_CODE" -ge 300 ]; then
                                    echo ""
                                    echo "ERROR: Policy update failed in ${target.name}."
                                    exit 1
                                fi

                                echo ""
                                echo "${target.name} policy updated successfully."
                            """


                            // =================================================
                            // ENVIRONMENT COMPLETE
                            // =================================================

                            echo ""
                            echo "=============================================="
                            echo "${target.name} GOVERNANCE SYNCHRONIZATION PASSED"
                            echo "=============================================="
                        }
                    }
                }
            }
        }
    }


    // =========================================================
    // POST ACTIONS
    // =========================================================

    post {

        success {

            echo '''
=================================================
 AXIAN CENTRAL GOVERNANCE RELEASE SUCCESS
=================================================

Governance rules have been successfully
synchronized across:

  ✓ Group
  ✓ Mvola
  ✓ Tigo

The central Git repository is the source
of truth for API governance.

=================================================
'''
        }

        failure {

            echo '''
=================================================
 AXIAN CENTRAL GOVERNANCE RELEASE FAILED
=================================================

Governance synchronization failed.

At least one target environment could not
be synchronized.

API governance release should be considered
BLOCKED until the issue is resolved.

=================================================
'''
        }
    }
}