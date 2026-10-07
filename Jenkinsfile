pipeline {
    agent any
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
                    ls -la
                    node --version
                    npm --version

                    # --- Network / DNS connectivity probe ---------------------------------
                    # The npm debug log showed every fetch failing with EAI_AGAIN
                    # (getaddrinfo on registry.npmjs.org), i.e. the container cannot
                    # resolve DNS. This probe makes that explicit in the console and
                    # fails fast with a clear message instead of the misleading
                    # "Exit handler never called!" crash from npm.
                    echo "=== Checking DNS / registry connectivity ==="
                    cat /etc/resolv.conf || true
                    if getent hosts registry.npmjs.org; then
                        echo "DNS OK: registry.npmjs.org resolved."
                    else
                        echo "DNS resolution FAILED for registry.npmjs.org (EAI_AGAIN root cause)."
                        echo "Fix Docker/WSL2 DNS (e.g. /etc/docker/daemon.json dns: [8.8.8.8,1.1.1.1]) or configure the corporate proxy, then re-run."
                        exit 1
                    fi
                    # Confirm we can actually reach the registry over HTTPS too.
                    npm ping || { echo "npm ping failed: registry reachable? check proxy/firewall."; exit 1; }
                    # ----------------------------------------------------------------------

                    # Start from a clean tree: a stale/corrupt node_modules from a
                    # previous crashed run can make npm ci choke and leaves bins missing.
                    rm -rf node_modules

                    npm ci --cache .npm --prefer-offline

                    # npm ci can crash in its exit handler yet still report success,
                    # so verify the install actually produced a usable tree.
                    if [ ! -x node_modules/.bin/react-scripts ]; then
                        echo "=== npm ci did not produce a working node_modules, dumping debug log ==="
                        cat .npm/_logs/*-debug-0.log 2>/dev/null
                        cat /home/node/.npm/_logs/*-debug-0.log 2>/dev/null
                        exit 1
                    fi

                    npm run build
                    ls -la
                
                '''
            }
        }
    }
}