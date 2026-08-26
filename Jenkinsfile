pipeline {

    agent any

    environment {

        // =========================================================
        // AXIAN APIM ENVIRONMENTS
        // =========================================================

        MVOLA_APIM_URL = 'https://13.49.18.52:9443'
        GROUP_APIM_URL = 'https://13.60.222.145:9443'
        TIGO_APIM_URL  = 'https://13.62.128.136:9443'
    }

    stages {

        // =========================================================
        // CHECKOUT CENTRAL GOVERNANCE REPOSITORY
        // =========================================================

        stage('Checkout Governance') {

            steps {

                checkout scm

                sh '''
                    set -e

                    echo "=============================================="
                    echo "        AXIAN CENTRAL API GOVERNANCE"
                    echo "=============================================="

                    echo ""
                    echo "Governance Version:"
                    cat version

                    echo ""
                    echo "Rules:"
                    ls -lh rules/

                    echo ""
                    echo "Governance package:"
                    find . -maxdepth 2 -type f | sort
                '''
            }
        }


        // =========================================================
        // VALIDATE GOVERNANCE PACKAGE
        // =========================================================

        stage('Validate Governance Package') {

            steps {

                sh '''
                    set -e

                    echo "=============================================="
                    echo "       VALIDATING GOVERNANCE PACKAGE"
                    echo "=============================================="

                    test -f version
                    test -f rules/group-api-standards.yaml

                    echo ""
                    echo "Governance version:"
                    cat version

                    echo ""
                    echo "Ruleset file:"
                    ls -lh rules/group-api-standards.yaml

                    echo ""
                    echo "Governance package validation PASSED."
                '''
            }
        }


        // =========================================================
        // DEPLOY CENTRAL GOVERNANCE
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
                        echo "================================================="
                        echo "        ${target.name} GOVERNANCE DEPLOYMENT"
                        echo "================================================="
                        echo "APIM URL : ${target.url}"
                        echo "Ruleset  : ${target.rulesetName}"
                        echo "Policy   : ${target.policyName}"
                        echo "================================================="


                        // =================================================
                        // CREDENTIALS
                        // =================================================

                        withCredentials([
                            usernamePassword(
                                credentialsId: target.credentialId,
                                usernameVariable: 'APIM_USER',
                                passwordVariable: 'APIM_PASS'
                            )
                        ]) {


                            // =================================================
                            // PASS TARGET INFORMATION TO SHELL
                            // =================================================

                            withEnv([
                                "TARGET_NAME=${target.name}",
                                "TARGET_URL=${target.url}",
                                "RULESET_NAME=${target.rulesetName}",
                                "POLICY_NAME=${target.policyName}"
                            ]) {


                                // =================================================
                                // 1. TEST CONNECTION
                                // =================================================

                                sh '''
                                    set -e

                                    echo ""
                                    echo "----------------------------------------------"
                                    echo "1. Testing APIM connection"
                                    echo "----------------------------------------------"

                                    HTTP_CODE=$(curl -sk \
                                        -o /tmp/${TARGET_NAME}-connection.json \
                                        -w "%{http_code}" \
                                        -u "$APIM_USER:$APIM_PASS" \
                                        -H "Accept: application/json" \
                                        "$TARGET_URL/api/am/governance/v1/rulesets")

                                    echo "HTTP Status: $HTTP_CODE"

                                    if [ "$HTTP_CODE" != "200" ]; then

                                        echo ""
                                        echo "ERROR: Unable to connect to $TARGET_NAME APIM."

                                        cat /tmp/${TARGET_NAME}-connection.json

                                        exit 1
                                    fi

                                    echo ""
                                    echo "$TARGET_NAME APIM connection successful."
                                '''


                                // =================================================
                                // 2. FIND RULESET
                                // =================================================

                                sh '''
                                    set -e

                                    echo ""
                                    echo "----------------------------------------------"
                                    echo "2. Finding Governance Ruleset"
                                    echo "----------------------------------------------"

                                    curl -sk \
                                        -u "$APIM_USER:$APIM_PASS" \
                                        -H "Accept: application/json" \
                                        "$TARGET_URL/api/am/governance/v1/rulesets?limit=100" \
                                        > ${TARGET_NAME}-rulesets.json

                                    echo ""
                                    echo "Rulesets available in $TARGET_NAME:"

                                    jq '
                                        .list[] |
                                        {
                                            id,
                                            name,
                                            ruleType,
                                            artifactType
                                        }
                                    ' ${TARGET_NAME}-rulesets.json

                                    RULESET_ID=$(jq -r \
                                        --arg NAME "$RULESET_NAME" \
                                        '
                                        .list[]
                                        | select(.name == $NAME)
                                        | .id
                                        ' \
                                        ${TARGET_NAME}-rulesets.json |
                                        head -n 1
                                    )

                                    if [ -z "$RULESET_ID" ]; then

                                        echo ""
                                        echo "Ruleset does not exist."
                                        echo "Action: CREATE"

                                        echo "CREATE" > ${TARGET_NAME}-ruleset-action

                                    else

                                        echo ""
                                        echo "Ruleset found."
                                        echo "Ruleset ID: $RULESET_ID"
                                        echo "Action: UPDATE"

                                        echo "$RULESET_ID" \
                                            > ${TARGET_NAME}-ruleset-id

                                        echo "UPDATE" \
                                            > ${TARGET_NAME}-ruleset-action

                                    fi
                                '''


                                // =================================================
                                // 3. CREATE RULESET
                                // =================================================

                                sh '''
                                    set -e

                                    ACTION=$(cat ${TARGET_NAME}-ruleset-action)

                                    if [ "$ACTION" != "CREATE" ]; then

                                        echo ""
                                        echo "Ruleset already exists."
                                        echo "Skipping CREATE."

                                        exit 0
                                    fi

                                    echo ""
                                    echo "----------------------------------------------"
                                    echo "3. Creating Governance Ruleset"
                                    echo "----------------------------------------------"

                                    HTTP_CODE=$(curl -sk \
                                        -u "$APIM_USER:$APIM_PASS" \
                                        -o ${TARGET_NAME}-ruleset-created.json \
                                        -w "%{http_code}" \
                                        -X POST \
                                        "$TARGET_URL/api/am/governance/v1/rulesets" \
                                        -H "Accept: application/json" \
                                        -F "name=$RULESET_NAME" \
                                        -F "description=Axian Group API Governance Standards" \
                                        -F "ruleType=API_METADATA" \
                                        -F "artifactType=REST_API" \
                                        -F "ruleCategory=SPECTRAL" \
                                        -F "rulesetContent=@rules/group-api-standards.yaml"
                                    )

                                    echo ""
                                    echo "HTTP Status: $HTTP_CODE"

                                    echo ""
                                    echo "Create response:"

                                    jq . ${TARGET_NAME}-ruleset-created.json \
                                        || cat ${TARGET_NAME}-ruleset-created.json

                                    if [ "$HTTP_CODE" -lt 200 ] || [ "$HTTP_CODE" -ge 300 ]; then

                                        echo ""
                                        echo "ERROR: Ruleset creation failed in $TARGET_NAME."

                                        exit 1
                                    fi

                                    RULESET_ID=$(jq -r \
                                        '.id // empty' \
                                        ${TARGET_NAME}-ruleset-created.json
                                    )

                                    if [ -z "$RULESET_ID" ]; then

                                        echo ""
                                        echo "ERROR: Ruleset ID was not returned."

                                        exit 1
                                    fi

                                    echo "$RULESET_ID" \
                                        > ${TARGET_NAME}-ruleset-id

                                    echo ""
                                    echo "Ruleset created successfully."
                                    echo "Ruleset ID: $RULESET_ID"
                                '''


                                // =================================================
                                // 4. UPDATE RULESET
                                // =================================================

                                sh '''
                                    set -e

                                    ACTION=$(cat ${TARGET_NAME}-ruleset-action)

                                    if [ "$ACTION" != "UPDATE" ]; then

                                        echo ""
                                        echo "Ruleset was created."
                                        echo "Skipping UPDATE."

                                        exit 0
                                    fi

                                    RULESET_ID=$(cat ${TARGET_NAME}-ruleset-id)

                                    echo ""
                                    echo "----------------------------------------------"
                                    echo "4. Updating Governance Ruleset"
                                    echo "----------------------------------------------"

                                    echo "Ruleset ID: $RULESET_ID"

                                    HTTP_CODE=$(curl -sk \
                                        -u "$APIM_USER:$APIM_PASS" \
                                        -o ${TARGET_NAME}-ruleset-updated.json \
                                        -w "%{http_code}" \
                                        -X PUT \
                                        "$TARGET_URL/api/am/governance/v1/rulesets/$RULESET_ID" \
                                        -H "Accept: application/json" \
                                        -F "name=$RULESET_NAME" \
                                        -F "description=Axian Group API Governance Standards" \
                                        -F "ruleType=API_METADATA" \
                                        -F "artifactType=REST_API" \
                                        -F "ruleCategory=SPECTRAL" \
                                        -F "rulesetContent=@rules/group-api-standards.yaml"
                                    )

                                    echo ""
                                    echo "HTTP Status: $HTTP_CODE"

                                    echo ""
                                    echo "Update response:"

                                    jq . ${TARGET_NAME}-ruleset-updated.json \
                                        || cat ${TARGET_NAME}-ruleset-updated.json

                                    if [ "$HTTP_CODE" -lt 200 ] || [ "$HTTP_CODE" -ge 300 ]; then

                                        echo ""
                                        echo "ERROR: Ruleset update failed in $TARGET_NAME."

                                        exit 1
                                    fi

                                    echo ""
                                    echo "Ruleset updated successfully."
                                '''


                                // =================================================
                                // 5. VERIFY RULESET
                                // =================================================

                                sh '''
                                    set -e

                                    RULESET_ID=$(cat ${TARGET_NAME}-ruleset-id)

                                    echo ""
                                    echo "----------------------------------------------"
                                    echo "5. Verifying Governance Ruleset"
                                    echo "----------------------------------------------"

                                    HTTP_CODE=$(curl -sk \
                                        -u "$APIM_USER:$APIM_PASS" \
                                        -o ${TARGET_NAME}-ruleset-verify.json \
                                        -w "%{http_code}" \
                                        -H "Accept: application/json" \
                                        "$TARGET_URL/api/am/governance/v1/rulesets/$RULESET_ID"
                                    )

                                    echo "HTTP Status: $HTTP_CODE"

                                    if [ "$HTTP_CODE" != "200" ]; then

                                        echo ""
                                        echo "ERROR: Ruleset verification failed."

                                        cat ${TARGET_NAME}-ruleset-verify.json

                                        exit 1
                                    fi

                                    jq . ${TARGET_NAME}-ruleset-verify.json

                                    NAME=$(jq -r \
                                        '.name // empty' \
                                        ${TARGET_NAME}-ruleset-verify.json
                                    )

                                    if [ "$NAME" != "$RULESET_NAME" ]; then

                                        echo ""
                                        echo "ERROR: Ruleset name verification failed."

                                        exit 1
                                    fi

                                    echo ""
                                    echo "Ruleset verification PASSED."
                                '''


                                // =================================================
                                // 6. FIND GOVERNANCE POLICY
                                // =================================================

                                sh '''
                                    set -e

                                    echo ""
                                    echo "----------------------------------------------"
                                    echo "6. Finding Governance Policy"
                                    echo "----------------------------------------------"

                                    curl -sk \
                                        -u "$APIM_USER:$APIM_PASS" \
                                        -H "Accept: application/json" \
                                        "$TARGET_URL/api/am/governance/v1/policies?limit=100" \
                                        > ${TARGET_NAME}-policies.json

                                    echo ""
                                    echo "Policies available in $TARGET_NAME:"

                                    jq '
                                        .list[] |
                                        {
                                            id,
                                            name
                                        }
                                    ' ${TARGET_NAME}-policies.json

                                    POLICY_ID=$(jq -r \
                                        --arg NAME "$POLICY_NAME" \
                                        '
                                        .list[]
                                        | select(.name == $NAME)
                                        | .id
                                        ' \
                                        ${TARGET_NAME}-policies.json |
                                        head -n 1
                                    )

                                    if [ -z "$POLICY_ID" ]; then

                                        echo ""
                                        echo "ERROR: Policy '$POLICY_NAME' not found in $TARGET_NAME."

                                        exit 1
                                    fi

                                    echo "$POLICY_ID" \
                                        > ${TARGET_NAME}-policy-id

                                    echo ""
                                    echo "Policy found:"
                                    echo "$POLICY_ID"
                                '''


                                // =================================================
                                // 7. GET EXISTING POLICY
                                // =================================================

                                sh '''
                                    set -e

                                    POLICY_ID=$(cat ${TARGET_NAME}-policy-id)

                                    echo ""
                                    echo "----------------------------------------------"
                                    echo "7. Getting Existing Governance Policy"
                                    echo "----------------------------------------------"

                                    HTTP_CODE=$(curl -sk \
                                        -u "$APIM_USER:$APIM_PASS" \
                                        -o ${TARGET_NAME}-policy-current.json \
                                        -w "%{http_code}" \
                                        -H "Accept: application/json" \
                                        "$TARGET_URL/api/am/governance/v1/policies/$POLICY_ID"
                                    )

                                    echo "HTTP Status: $HTTP_CODE"

                                    if [ "$HTTP_CODE" != "200" ]; then

                                        echo ""
                                        echo "ERROR: Unable to retrieve governance policy."

                                        cat ${TARGET_NAME}-policy-current.json

                                        exit 1
                                    fi

                                    jq . ${TARGET_NAME}-policy-current.json
                                '''


                                // =================================================
                                // 8. ASSOCIATE RULESET WITH POLICY
                                // =================================================

                                sh '''
                                    set -e

                                    POLICY_ID=$(cat ${TARGET_NAME}-policy-id)
                                    RULESET_ID=$(cat ${TARGET_NAME}-ruleset-id)

                                    echo ""
                                    echo "----------------------------------------------"
                                    echo "8. Associating Ruleset With Policy"
                                    echo "----------------------------------------------"

                                    echo "Policy ID : $POLICY_ID"
                                    echo "Ruleset ID: $RULESET_ID"

                                    jq \
                                        --arg ruleset "$RULESET_ID" \
                                        '
                                        .rulesets =
                                        (
                                            (.rulesets // [])
                                            + [$ruleset]
                                            | unique
                                        )
                                        ' \
                                        ${TARGET_NAME}-policy-current.json \
                                        > ${TARGET_NAME}-policy-updated.json

                                    echo ""
                                    echo "Updated policy:"

                                    jq . ${TARGET_NAME}-policy-updated.json


                                    HTTP_CODE=$(curl -sk \
                                        -u "$APIM_USER:$APIM_PASS" \
                                        -o ${TARGET_NAME}-policy-updated-response.json \
                                        -w "%{http_code}" \
                                        -X PUT \
                                        "$TARGET_URL/api/am/governance/v1/policies/$POLICY_ID" \
                                        -H "Accept: application/json" \
                                        -H "Content-Type: application/json" \
                                        --data @${TARGET_NAME}-policy-updated.json
                                    )

                                    echo ""
                                    echo "HTTP Status: $HTTP_CODE"

                                    echo ""
                                    echo "Policy update response:"

                                    jq . ${TARGET_NAME}-policy-updated-response.json \
                                        || cat ${TARGET_NAME}-policy-updated-response.json


                                    if [ "$HTTP_CODE" -lt 200 ] || [ "$HTTP_CODE" -ge 300 ]; then

                                        echo ""
                                        echo "ERROR: Policy update failed in $TARGET_NAME."

                                        exit 1
                                    fi

                                    echo ""
                                    echo "Policy updated successfully."
                                '''


                                // =================================================
                                // 9. VERIFY POLICY
                                // =================================================

                                sh '''
                                    set -e

                                    POLICY_ID=$(cat ${TARGET_NAME}-policy-id)

                                    echo ""
                                    echo "----------------------------------------------"
                                    echo "9. Verifying Governance Policy"
                                    echo "----------------------------------------------"

                                    HTTP_CODE=$(curl -sk \
                                        -u "$APIM_USER:$APIM_PASS" \
                                        -o ${TARGET_NAME}-policy-verify.json \
                                        -w "%{http_code}" \
                                        -H "Accept: application/json" \
                                        "$TARGET_URL/api/am/governance/v1/policies/$POLICY_ID"
                                    )

                                    echo "HTTP Status: $HTTP_CODE"

                                    if [ "$HTTP_CODE" != "200" ]; then

                                        echo ""
                                        echo "ERROR: Policy verification failed."

                                        cat ${TARGET_NAME}-policy-verify.json

                                        exit 1
                                    fi

                                    jq . ${TARGET_NAME}-policy-verify.json

                                    echo ""
                                    echo "Policy verification PASSED."
                                '''


                                // =================================================
                                // ENVIRONMENT SUCCESS
                                // =================================================

                                echo ""
                                echo "================================================="
                                echo " ${target.name} GOVERNANCE SYNCHRONIZATION PASSED"
                                echo "================================================="
                            }
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
========================================================
       AXIAN CENTRAL GOVERNANCE - SUCCESS
========================================================

Central governance has been synchronized successfully.

Environments:

    ✓ GROUP
    ✓ MVOLA
    ✓ TIGO

The GitHub governance repository remains the
central source of truth.

========================================================
'''
        }


        failure {

            echo '''
========================================================
       AXIAN CENTRAL GOVERNANCE - FAILED
========================================================

Governance synchronization failed.

At least one environment could not be synchronized.

Governance deployment should be considered BLOCKED
until the failure is resolved.

========================================================
'''
        }
    }
}