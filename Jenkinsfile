pipeline {

    agent any

    stages {

        stage('Checkout Governance') {
            steps {
                checkout scm

                sh '''
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
    }
}
