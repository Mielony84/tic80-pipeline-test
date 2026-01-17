pipeline {
    agent {
        docker {
            image 'tic80-builder:latest'
            args '-u root:root'
        }
    }

    environment {
        VERSION = "test-version-3"
        RELEASE_NAME = "TIC-80 test version 3"
        RELEASE_BODY = "Automatyczny release z Jenkinsa (Linux x86_64)"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Clone TIC-80') {
            steps {
                sh '''
                    rm -rf TIC-80
                    git clone https://github.com/nesbox/TIC-80.git
                    cd TIC-80
                    git submodule update --init --recursive
                '''
            }
        }

        stage('Build TIC-80 (Linux x86_64)') {
            steps {
                sh '''
                    cd TIC-80
                    mkdir -p build
                    cd build
                    cmake ..
                    make -j$(nproc)
                '''
            }
        }

        stage('Package (Linux x86_64)') {
            steps {
                sh '''
                    cd TIC-80/build/bin
                    tar -czf tic80-linux-x86_64.tar.gz tic80
                    sha256sum tic80-linux-x86_64.tar.gz > tic80-linux-x86_64.sha256
                '''
            }
        }

        stage('Create GitHub Release') {
            steps {
                withCredentials([string(credentialsId: 'github-pat', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                        curl -X POST \
                          -H "Authorization: Bearer $GITHUB_TOKEN" \
                          -H "Content-Type: application/json" \
                          -d "{
                                \\"tag_name\\": \\"${VERSION}\\",
                                \\"name\\": \\"${RELEASE_NAME}\\",
                                \\"body\\": \\"${RELEASE_BODY}\\",
                                \\"draft\\": false,
                                \\"prerelease\\": false
                              }" \
                          https://api.github.com/repos/Mielony84/tic80-pipeline-test/releases
                    '''
                }
            }
        }

        stage('Upload Linux artifacts') {
            steps {
                withCredentials([string(credentialsId: 'github-pat', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                        RELEASE_ID=$(curl -s \
                          -H "Authorization: Bearer $GITHUB_TOKEN" \
                          https://api.github.com/repos/Mielony84/tic80-pipeline-test/releases/tags/${VERSION} \
                          | jq -r .id)

                        echo "Release ID: $RELEASE_ID"

                        cd TIC-80/build/bin

                        curl -X POST \
                          -H "Authorization: Bearer $GITHUB_TOKEN" \
                          -H "Content-Type: application/octet-stream" \
                          --data-binary @tic80-linux-x86_64.tar.gz \
                          https://uploads.github.com/repos/Mielony84/tic80-pipeline-test/releases/$RELEASE_ID/assets?name=tic80-linux-x86_64.tar.gz

                        curl -X POST \
                          -H "Authorization: Bearer $GITHUB_TOKEN" \
                          -H "Content-Type: application/octet-stream" \
                          --data-binary @tic80-linux-x86_64.sha256 \
                          https://uploads.github.com/repos/Mielony84/tic80-pipeline-test/releases/$RELEASE_ID/assets?name=tic80-linux-x86_64.sha256
                    '''
                }
            }
        }
    }
}
