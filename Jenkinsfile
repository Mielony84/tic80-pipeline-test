pipeline {
    agent any

    environment {
        GITHUB_TOKEN = credentials('github-pat')
        REPO = "Mielony84/tic80-pipeline-test"
        TAG = "test-version-${BUILD_NUMBER}"
        RELEASE_NAME = "TIC-80 test version ${BUILD_NUMBER}"
        ARTIFACT_TAR = "tic80-linux-x86_64.tar.gz"
        ARTIFACT_SHA = "tic80-linux-x86_64.sha256"
    }

    stages {

        stage('Prepare environment') {
            steps {
                sh '''
                sudo apt update
                sudo apt install -y \
                  build-essential cmake libsdl2-dev libreadline-dev \
                  libcurl4-openssl-dev libgtk-3-dev libgles2-mesa-dev \
                  liblua5.3-dev zlib1g-dev
                '''
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
                tar -czf ${ARTIFACT_TAR} tic80
                sha256sum ${ARTIFACT_TAR} > ${ARTIFACT_SHA}
                '''
            }
        }

        stage('Create GitHub Release') {
            steps {
                sh '''
                curl -X POST \
                  -H "Authorization: token ${GITHUB_TOKEN}" \
                  -H "Content-Type: application/json" \
                  -d "{
                        \\"tag_name\\": \\"${TAG}\\",
                        \\"name\\": \\"${RELEASE_NAME}\\",
                        \\"body\\": \\"Automatyczny release z Jenkinsa (Linux x86_64)\\",
                        \\"draft\\": false,
                        \\"prerelease\\": false
                      }" \
                  https://api.github.com/repos/${REPO}/releases
                '''
            }
        }

        stage('Upload Linux artifacts') {
            steps {
                sh '''
                RELEASE_ID=$(curl -s \
                  -H "Authorization: token ${GITHUB_TOKEN}" \
                  https://api.github.com/repos/${REPO}/releases/tags/${TAG} \
                  | jq -r '.id')

                cd TIC-80/build/bin

                curl -X POST \
                  -H "Authorization: token ${GITHUB_TOKEN}" \
                  -H "Content-Type: application/octet-stream" \
                  --data-binary @${ARTIFACT_TAR} \
                  "https://uploads.github.com/repos/${REPO}/releases/${RELEASE_ID}/assets?name=${ARTIFACT_TAR}"

                curl -X POST \
                  -H "Authorization: token ${GITHUB_TOKEN}" \
                  -H "Content-Type: application/octet-stream" \
                  --data-binary @${ARTIFACT_SHA} \
                  "https://uploads.github.com/repos/${REPO}/releases/${RELEASE_ID}/assets?name=${ARTIFACT_SHA}"
                '''
            }
        }
    }
}
