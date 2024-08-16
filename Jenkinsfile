pipeline {
    agent any
    
    environment {
        ARTIFACT_PATH = '/home/vagrant/lila/target'
        BUILD_SERVER = 'vagrant@192.168.0.2'
        DEFAULT_VERSION = '1.0.5'
        VERSION = "${DEFAULT_VERSION}"
        ARTIFACT_FILE = 'lila_${VERSION}_all.deb'
        GITHUB_REPO = 'tarashoshko/lila'
        GIT_BRANCH = 'main'
        GITHUB_TOKEN = credentials('Github_token')
        GITHUB_CREDENTIALS_ID = 'GIT_SSH'
        SBIN_PATH = '/home/vagrant/.local/share/coursier/bin'
        ORCHESTRATOR_HOST = '10.0.0.6'
        ORCHESTRATOR_USER = 'vagrant'
        SSH = credentials('SSH_PRIVATE_KEY')
        VAULT_PASS = credentials('token_password')
        
    }
    triggers {
        pollSCM('H/5 * * * *')
    }
    stages {
        stage('Checkout') {
            steps {
                script {
                    def changes = sh(script: 'git diff --name-only HEAD HEAD~1', returnStdout: true).trim()
                    if (!changes) {
                        currentBuild.result = 'SUCCESS'
                        echo "No changes detected. Skipping build."
                        currentBuild.rawBuild.getExecutor().interrupt(Result.ABORTED)
                        return
                    }
                }
            }
        }
        stage('Setup SSH Known Hosts') {
            steps {
                script {
                    sh '''
                    mkdir -p ~/.ssh
                    ssh-keyscan -t ecdsa github.com >> ~/.ssh/known_hosts
                    chmod 644 ~/.ssh/known_hosts
                    '''
                }
            }
        }
        stage('Upload GitHub Token to Orchestrator') {
            steps {
                script {
                    sh '''
                    echo $GITHUB_TOKEN > /tmp/github_token.txt
                    scp -i ${SSH} -o StrictHostKeyChecking=no /tmp/github_token.txt ${ORCHESTRATOR_USER}@${ORCHESTRATOR_HOST}:/vagrant/ansible/playbooks/roles/development/files/github_token.txt
                    '''
                }
            }
        }
        stage('Configure Jenkins Agent via Orchestrator') {
            steps {
                script {
                    sh '''
                    echo ${VAULT_PASS} > /tmp/vault_password.txt
                    scp -i ${SSH} -o StrictHostKeyChecking=no /tmp/vault_password.txt ${ORCHESTRATOR_USER}@${ORCHESTRATOR_HOST}:/tmp/vault_password.txt
                    ssh -i ${SSH} -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${ORCHESTRATOR_USER}@${ORCHESTRATOR_HOST} "
                        cd /${ORCHESTRATOR_USER}/ansible && ansible-playbook -i inventory.ini playbooks/development.yml --vault-password-file /tmp/vault_password.txt
                    "
                    ssh -i ${SSH} -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${ORCHESTRATOR_USER}@${ORCHESTRATOR_HOST} "
                        rm /tmp/vault_password.txt
                    "
                    '''
                }
            }
        }
        stage('Determine Version') {
            steps {
                script {
                    try {
                        withCredentials([sshUserPrivateKey(credentialsId: "${GITHUB_CREDENTIALS_ID}", keyFileVariable: 'GIT_SSH')]) {
                            sh 'GIT_SSH_COMMAND="ssh -i ${GIT_SSH}" git fetch --tags'
                            
                            def latestTag = sh(script: 'git describe --tags --abbrev=0 || echo "no-tags"', returnStdout: true).trim()
                            echo "Latest Tag: ${latestTag}"
                            echo "Current VERSION: ${env.VERSION}"
        
                            if (latestTag != "no-tags") {
                                def versionParts = latestTag.tokenize('.')
                                def major = versionParts[0].toInteger()
                                def minor = versionParts[1].toInteger()
                                def patch = versionParts[2].toInteger()
        
                                env.VERSION = "${major}.${minor}.${patch}"
                                env.ARTIFACT_FILE = "lila_${env.VERSION}_all.deb"
                            } else {
                                error 'No tags found, using default version.'
                                return
                            }
                        }
                    } catch (Exception e) {
                        echo "Error retrieving version: ${e.message}, setting default version."
                        env.VERSION = DEFAULT_VERSION
                        env.ARTIFACT_FILE = "lila_${env.VERSION}_all.deb"
                    }
                    echo "Version: ${env.VERSION}"
                }
            }
        }
        stage('Build UI') {
            agent { label 'agent1' }
            steps {
                script {
                    sh """
                    /home/vagrant/lila/ui/build
                    """
                }
            }
        }
        stage('Build App') {
            agent { label 'agent1' }
            steps {
                script {
                    sh """
                    export PATH=\$PATH:${SBIN_PATH} 
                    cd /home/vagrant/lila 
                    sbt -DVERSION=${env.VERSION} compile debian:packageBin
                    """
                }
            }
        }
        stage('Upload to GitHub Releases') {
            agent { label 'agent1' }
            steps {
                script {
                    withCredentials([string(credentialsId: 'GITHUB_TOKEN', variable: 'GITHUB_TOKEN')]) {
                        def releaseUrl = "https://api.github.com/repos/${GITHUB_REPO}/releases"
                        def releaseData = """{
                            "tag_name": "${env.VERSION}",
                            "name": "Release ${env.VERSION}",
                            "body": "Release notes"
                        }"""
                        
                        echo "Release data: ${releaseData}"
        
                        def existingRelease = sh(script: """
                            curl -H "Authorization: token \$GITHUB_TOKEN" \
                                 -H "Accept: application/vnd.github.v3+json" \
                                 ${releaseUrl}?per_page=100 | grep -o '"tag_name": "'${VERSION}'"' || true
                        """, returnStdout: true).trim()
                        
                        echo "Existing release: ${existingRelease}"
                        
                        if (!existingRelease) {
                            def createReleaseResponse = sh(script: """
                                curl -H "Authorization: token \$GITHUB_TOKEN" \
                                     -H "Accept: application/vnd.github.v3+json" \
                                     -X POST \
                                     -d '${releaseData}' \
                                     ${releaseUrl}
                            """, returnStdout: true).trim()
        
                            def releaseId = sh(script: """
                                echo '${createReleaseResponse}' | jq -r '.id'
                            """, returnStdout: true).trim()
        
                            if (!releaseId) {
                                error "Failed to extract releaseId from response."
                            }
        
                            echo "New Release ID: ${releaseId}"
        
                            def uploadUrl = "https://uploads.github.com/repos/${GITHUB_REPO}/releases/${releaseId}/assets?name=${ARTIFACT_FILE}"
        
                            echo "Upload URL: ${uploadUrl}"
        
                            sh """
                            curl -H "Authorization: token \$GITHUB_TOKEN" \
                                 -H "Content-Type: application/octet-stream" \
                                 --data-binary @/home/vagrant/lila/target/${ARTIFACT_FILE} \
                                 "${uploadUrl}"
                            """
                        } else {
                            echo "Release with tag '${VERSION}' already exists."
                        }
                    }
                }
            }
        }
        stage('Download Artifact to Orchestrator') {
            agent { label 'agent1' }
            steps {
                script {
                    sshagent(credentials: ['SSH_PRIVATE_KEY']) {
                        sh """
                        scp -o StrictHostKeyChecking=no /home/vagrant/lila/target/${ARTIFACT_FILE} ${ORCHESTRATOR_USER}@${ORCHESTRATOR_HOST}:/home/${ORCHESTRATOR_USER}/${ARTIFACT_FILE}
                        """
                    }
                }
            }
        }
        stage('Deploy') {
            steps {
                script {
                    sh """
                    ssh -i ${SSH} -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${ORCHESTRATOR_USER}@${ORCHESTRATOR_HOST} "
                        cd /${ORCHESTRATOR_USER}/ansible &&
                        ansible-playbook -i inventory.ini playbooks/application.yml -e 'version=${VERSION} artifact_file=${ARTIFACT_FILE}'
                    "
                    """
                }
            }
        }
        stage('Test') {
            steps {
                script {
                    echo "Running  test..."
                    
                    if (0 == 0) {
                        echo "Test passed."
                    }
                }
            }
        }
    }
}
