pipeline {
    agent any

    options {
        // The EAI_AGAIN failures were transient DNS hiccups from the Docker Desktop /
        // Rancher internal resolver. Retry the whole build rather than hand-babysitting it.
        retry(2)
        timeout(time: 30, unit: 'MINUTES')
    }

    stages {
        stage('Build'){
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps{
                sh '''
                    set -e

                    node --version
                    npm --version

                    # --- Connectivity probe ----------------------------------------------
                    # Diagnostic only: never hard-fail here. DNS on this host comes from the
                    # Docker Desktop internal resolver (192.168.127.1) and is occasionally
                    # slow to come up. Do NOT pin external DNS (8.8.8.8 / 1.1.1.1) -- it is
                    # blocked on this corporate network and resolution fails outright.
                    echo "=== DNS / registry connectivity ==="
                    cat /etc/resolv.conf 2>/dev/null || true
                    for i in 1 2 3 4 5 6; do
                        if getent hosts registry.npmjs.org >/dev/null 2>&1; then
                            echo "DNS OK (attempt $i): $(getent hosts registry.npmjs.org)"
                            break
                        fi
                        echo "DNS not ready (attempt $i/6), waiting 5s..."
                        sleep 5
                    done
                    # ----------------------------------------------------------------------

                    # Harden npm against the flaky resolver. --no-audit is the important one:
                    # the audit bulk request was what threw FetchError and tipped npm into its
                    # bogus "Exit handler never called!" crash.
                    npm config set fetch-retries 5
                    npm config set fetch-retry-mintimeout 20000
                    npm config set fetch-retry-maxtimeout 120000
                    npm config set fetch-timeout 300000

                    # Start from a clean tree: a stale/partial node_modules from a previously
                    # crashed run leaves binaries like react-scripts missing.
                    rm -rf node_modules

                    # Retry npm ci itself, since a transient DNS blip kills the whole install.
                    INSTALL_OK=0
                    for attempt in 1 2 3; do
                        echo "=== npm ci attempt $attempt/3 ==="
                        npm ci --no-audit --no-fund --cache .npm --prefer-offline || true

                        # npm ci can crash in its exit handler yet still exit 0, so trust the
                        # filesystem rather than the exit code.
                        if [ -x node_modules/.bin/react-scripts ]; then
                            echo "npm ci produced a usable node_modules."
                            INSTALL_OK=1
                            break
                        fi

                        echo "Install incomplete (react-scripts missing). Retrying in 15s..."
                        rm -rf node_modules
                        sleep 15
                    done

                    if [ "$INSTALL_OK" -ne 1 ]; then
                        echo "=== npm ci failed after 3 attempts, dumping debug log ==="
                        tail -50 .npm/_logs/*-debug-0.log 2>/dev/null || true
                        exit 1
                    fi

                    npm run build

                    # Prove the build actually emitted artifacts.
                    test -f build/index.html || { echo "build/index.html missing - build did not produce output"; exit 1; }
                    echo "=== Build output ==="
                    ls -la build
                '''
            }
        }
    }
}
