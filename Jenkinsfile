#!/usr/bin/env groovy

commit_id = ''

try {
    stage('Checkout') {
        node('rippled-dev') {
            checkout scm
            commit_id = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
            echo "Parsurvan xrp build — commit: ${commit_id}"
        }
    }

    stage('UI') {
        node('rippled-dev') {
            checkout scm
            sh '''
                node --version
                cd bin
                for f in browser.js jsonrpc_server.js jsonrpc_request.js; do
                    [ -f "$f" ] && node --check "$f" && echo "OK: $f" || true
                done
            '''
        }
    }

    stage('GPG Commit') {
        node('rippled-dev') {
            withCredentials([string(credentialsId: 'gpg-signing-key-id', variable: 'GPG_KEY')]) {
                sh """
                    git config user.signingkey \${GPG_KEY}
                    git config commit.gpgsign true
                    git tag -s "build-\${BUILD_NUMBER}" \
                        -m "ci(xrp): build #\${BUILD_NUMBER} \${commit_id}"
                """
            }
        }
    }

    stage('Publish to S3') {
        node('rippled-dev') {
            withCredentials([
                string(credentialsId: 'contabo-s3-access-key', variable: 'S3_ACCESS'),
                string(credentialsId: 'contabo-s3-secret-key', variable: 'S3_SECRET')
            ]) {
                step([$class: 'PegatonCodeDeployPublisher',
                    ossBucket:    'pegaton',
                    ossObject:    'xrp-ci',
                    region:       'eu2',
                    includes:     'bin/browser.js,bin/*.js',
                    excludes:     '',
                    subdirectory: '',
                    accessKey:    env.S3_ACCESS,
                    secretKey:    env.S3_SECRET,
                    doDeploy:     false,
                    instanceId:   ''
                ])
            }
        }
    }
}
catch (e) {
    echo "Build failed: ${e}"
    throw e
}
finally {
    stage('Final Status') {
        node {
            echo "Result: ${currentBuild.currentResult} — ${commit_id}"
        }
    }
}
