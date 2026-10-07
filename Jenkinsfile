pipeline {
    agent any

    options {
        // Keep a hard ceiling so a network problem can never hang the executor for
        // half an hour again. Deliberately no retry() here: retrying a DNS failure
        // just multiplies the wait without changing the outcome.
        timeout(time: 15, unit: 'MINUTES')
    }

    stages {
        stage('Build'){
            agent {
                docker {
                    image 'node:18-alpine'
                    // THE FIX. Jenkins talks to the `docker:dind` daemon
                    // (DOCKER_HOST=tcp://docker:2376), and dind hands its child
                    // containers 8.8.8.8/8.8.4.4 by default. Those are blocked on this
                    // corporate network, so every npm fetch died with EAI_AGAIN.
                    // 192.168.127.1 is the Docker Desktop internal resolver, which is
                    // the only one that resolves here (verified: 8.8.8.8 fails, this works).
                    args '--dns 192.168.127.1'
                    reuseNode true
                }
            }
            steps{
                sh '''
                    set -e

                    node --version
                    npm --version

                    # --- Fail fast if DNS is broken --------------------------------------
                    # Without this, npm retries every one of ~1500 packages and the stage
                    # hangs for tens of minutes instead of telling you what is wrong.
                    echo "=== DNS check ==="
                    cat /etc/resolv.conf 2>/dev/null || true
                    if ! getent hosts registry.npmjs.org; then
                        echo "FATAL: cannot resolve registry.npmjs.org from this container."
                        echo "dind is probably handing out 8.8.8.8 again (blocked here)."
                        echo "Expected nameserver 192.168.127.1 via the agent's --dns arg."
                        exit 1
                    fi
                    echo "DNS OK."
                    # ----------------------------------------------------------------------

                    # Short, sane network settings. A real outage should surface in about a
                    # minute, not thirty.
                    npm config set fetch-retries 2
                    npm config set fetch-retry-maxtimeout 30000
                    npm config set fetch-timeout 60000

                    # --no-audit matters: the audit bulk request is what threw FetchError
                    # and tipped npm into its misleading "Exit handler never called!" crash.
                    rm -rf node_modules
                    npm ci --no-audit --no-fund

                    # npm can exit 0 even when it crashed, so trust the filesystem.
                    test -x node_modules/.bin/react-scripts \
                        || { echo "FATAL: npm ci finished but react-scripts is missing"; exit 1; }

                    npm run build

                    test -f build/index.html \
                        || { echo "FATAL: build produced no build/index.html"; exit 1; }

                    echo "=== Build output ==="
                    ls -la build
                '''
            }
        }
    }
}
