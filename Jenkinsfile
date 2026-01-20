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
        GITHUB_REPO = "Mielony84/tic80-pipeline-test"
    }

    stages {
        // STAGE CHECKOUT USUŃ - Jenkins już to zrobił przez SCM

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
                    # Ustaw cross-compilation
                    cmake .. \
                        -DCMAKE_C_COMPILER=x86_64-linux-gnu-gcc \
                        -DCMAKE_CXX_COMPILER=x86_64-linux-gnu-g++ \
                        -DCMAKE_SYSTEM_NAME=Linux \
                        -DCMAKE_SYSTEM_PROCESSOR=x86_64
                    make -j$(nproc)
                '''
            }
        }

        stage('Package (Linux x86_64)') {
            steps {
                sh '''
                    cd TIC-80/build/bin
                    tar -czf tic80-linux-x86_64.tar.gz tic80
                    sha256sum tic80-linux-x86_64.tar.gz | cut -d' ' -f1 > tic80-linux-x86_64.sha256
                '''
                archiveArtifacts artifacts: 'TIC-80/build/bin/tic80-linux-x86_64.*'
            }
        }

        stage('Create GitHub Release') {
            steps {
                withCredentials([string(credentialsId: 'github-pat', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                        # Najpierw sprawdź czy tag już istnieje
                        if curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
                           https://api.github.com/repos/${GITHUB_REPO}/releases/tags/${VERSION} \
                           | grep -q "Not Found"; then
                           
                            echo "Tworzenie nowego release..."
                            curl -X POST \
                              -H "Authorization: Bearer $GITHUB_TOKEN" \
                              -H "Content-Type: application/json" \
                              -d "{
                                    \"tag_name\": \"${VERSION}\",
                                    \"name\": \"${RELEASE_NAME}\",
                                    \"body\": \"${RELEASE_BODY}\",
                                    \"draft\": false,
                                    \"prerelease\": false
                                  }" \
                              https://api.github.com/repos/${GITHUB_REPO}/releases
                        else
                            echo "Release ${VERSION} już istnieje"
                        fi
                    '''
                }
            }
        }

        stage('Upload Linux artifacts') {
            when {
                expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
            }
            steps {
                withCredentials([string(credentialsId: 'github-pat', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                        # Pobierz ID release
                        RELEASE_ID=$(curl -s \
                          -H "Authorization: Bearer $GITHUB_TOKEN" \
                          https://api.github.com/repos/${GITHUB_REPO}/releases/tags/${VERSION} \
                          | jq -r .id)

                        echo "Release ID: $RELEASE_ID"

                        cd TIC-80/build/bin

                        # Prześlij główny plik
                        curl -X POST \
                          -H "Authorization: Bearer $GITHUB_TOKEN" \
                          -H "Content-Type: application/octet-stream" \
                          --data-binary @tic80-linux-x86_64.tar.gz \
                          "https://uploads.github.com/repos/${GITHUB_REPO}/releases/$RELEASE_ID/assets?name=tic80-linux-x86_64.tar.gz"

                        # Prześlij checksum
                        curl -X POST \
                          -H "Authorization: Bearer $GITHUB_TOKEN" \
                          -H "Content-Type: application/octet-stream" \
                          --data-binary @tic80-linux-x86_64.sha256 \
                          "https://uploads.github.com/repos/${GITHUB_REPO}/releases/$RELEASE_ID/assets?name=tic80-linux-x86_64.sha256"
                    '''
                }
            }
        }
    }
    
    post {
        always {
            cleanWs() // Wyczyść workspace po buildzie
        }
    }
}
