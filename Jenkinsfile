// Jenkins pipeline for mandi-automl.
//
// Build and test happen here; the actual deployment is delegated to the companion
// `devops_IA3` repository, which owns the Terraform and Ansible definitions.
//
// Every credential is referenced by its Jenkins credential ID. Nothing secret is
// written in this file.
//
// Jenkins runs locally (see docs/deployment.md), so a local install cannot receive
// a GitHub webhook. Poll SCM is used instead; the assignment allows either.

// A `when { branch ... }` directive only works in a Multibranch Pipeline or an
// Organization Folder, where Jenkins sets BRANCH_NAME. This job is a plain
// "Pipeline script from SCM", where BRANCH_NAME is null, so that directive
// silently skipped Push, Deploy and Verify while the build still went green --
// a false success, which is worse than a failure because nothing looks wrong.
//
// The git plugin does set GIT_BRANCH (as "origin/main"), so fall back to it and
// accept either spelling. This keeps working unchanged if the job is ever
// converted to Multibranch.
def onDeployBranch() {
    String branch = env.BRANCH_NAME ?: env.GIT_BRANCH ?: ''
    return branch == 'main' || branch.endsWith('/main')
}

pipeline {
    agent any

    triggers {
        // Ask GitHub every two minutes whether main moved. `H` spreads the load so
        // several jobs on one Jenkins do not all poll on the same second.
        pollSCM('H/2 * * * *')
    }

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timeout(time: 40, unit: 'MINUTES')
    }

    environment {
        REGISTRY     = 'ghcr.io'
        // IMAGE_NAME is NOT set here. It is derived in the Checkout stage from
        // whichever fork is being built, so a teammate building their own fork
        // pushes to their own GHCR namespace with their own token, instead of
        // needing write access to someone else's packages.
        IMAGE_TAG    = "${env.GIT_COMMIT ? env.GIT_COMMIT.take(7) : 'local'}"
        INFRA_REPO   = 'https://github.com/dharshan-kumarj/Devops_IA3.git'
        INFRA_BRANCH = 'main'
        UV_CACHE_DIR = "${env.WORKSPACE}/.uv-cache"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git rev-parse --short HEAD'
                script {
                    // Take the GitHub owner from the remote this build came from.
                    // GHCR rejects uppercase in a repository path, so the whole
                    // thing is lowercased -- which is why this is not simply the
                    // repo's own name (AutoML_IA3).
                    String url = sh(script: 'git config --get remote.origin.url',
                                    returnStdout: true).trim()
                    def match = url =~ /[:\/]([^\/:]+)\/[^\/]+?(?:\.git)?$/
                    if (!match) {
                        error "Could not read the GitHub owner from remote: ${url}"
                    }
                    env.IMAGE_NAME = "${env.REGISTRY}/${match[0][1].toLowerCase()}/automl_ia3"
                    echo "Building ${env.IMAGE_NAME}:${env.IMAGE_TAG}"
                }
            }
        }

        stage('Lint & Test') {
            steps {
                sh '''
                    set -eu
                    curl -LsSf https://astral.sh/uv/install.sh | sh
                    export PATH="$HOME/.local/bin:$PATH"
                    uv sync --frozen --no-default-groups --group dev
                    uv run ruff check .
                    uv run ruff format --check .
                    uv run pytest -q
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    set -eu
                    # The Dockerfile uses `RUN --mount=type=cache`, which the legacy
                    # builder cannot parse -- it fails with "the --mount option
                    # requires BuildKit" several steps in, after sending the whole
                    # build context. Fail up front with something actionable instead.
                    if ! docker buildx version >/dev/null 2>&1; then
                        echo "docker buildx is not installed, and this Dockerfile needs it."
                        echo "Install it with:   sudo apt install -y docker-buildx-plugin"
                        echo "then restart Jenkins:   sudo systemctl restart jenkins"
                        exit 1
                    fi
                    export DOCKER_BUILDKIT=1

                    # The image serves from models/ alone: the per-crop models and
                    # models/weekly_snapshot.csv, both committed. data/ is excluded by
                    # .dockerignore, so nothing under it is required to build here.
                    test -f models/index.json || {
                        echo "models/index.json is missing. Register models before building:"
                        echo "  uv run python -m automl.registry"
                        exit 1
                    }
                    test -f models/weekly_snapshot.csv || {
                        echo "models/weekly_snapshot.csv is missing. The API computes from it."
                        exit 1
                    }
                    docker build -t "${IMAGE_NAME}:${IMAGE_TAG}" -t "${IMAGE_NAME}:latest" .
                '''
            }
        }

        stage('Smoke Test Image') {
            steps {
                sh '''
                    set -eu
                    docker rm -f mandi-smoke >/dev/null 2>&1 || true
                    docker run -d --name mandi-smoke -p 18000:8000 "${IMAGE_NAME}:${IMAGE_TAG}"
                    for attempt in $(seq 1 30); do
                        if curl -fsS http://127.0.0.1:18000/api/health; then break; fi
                        sleep 2
                    done
                    curl -fsS http://127.0.0.1:18000/api/health | grep -q '"status":"ok"'
                '''
            }
            post {
                always {
                    sh 'docker rm -f mandi-smoke >/dev/null 2>&1 || true'
                }
            }
        }

        stage('Push to GHCR') {
            when { expression { onDeployBranch() } }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'ghcr-credentials',
                    usernameVariable: 'GHCR_USER',
                    passwordVariable: 'GHCR_TOKEN'
                )]) {
                    sh '''
                        set -eu
                        # Keep the registry credential in a throwaway config
                        # inside the workspace, instead of the jenkins user's
                        # shared ~/.docker/config.json.
                        #
                        # Two reasons. A failed push used to leave a bad GHCR
                        # credential behind -- `docker logout` sat after the
                        # push under `set -eu`, so it never ran -- and every
                        # later build then sent that broken token when pulling
                        # even PUBLIC ghcr.io images, failing with
                        # "failed to fetch oauth token: denied". And a token on
                        # disk outlives the build that needed it.
                        #
                        # DOCKER_CONFIG only moves client-side auth; the image
                        # itself lives in the daemon, so push still finds it.
                        export DOCKER_CONFIG="${WORKSPACE}/.docker-ghcr"
                        rm -rf "$DOCKER_CONFIG"
                        mkdir -p "$DOCKER_CONFIG"
                        trap 'rm -rf "$DOCKER_CONFIG"' EXIT INT TERM

                        echo "$GHCR_TOKEN" | docker login "${REGISTRY}" -u "$GHCR_USER" --password-stdin
                        docker push "${IMAGE_NAME}:${IMAGE_TAG}"
                        docker push "${IMAGE_NAME}:latest"
                    '''
                }
            }
        }

        stage('Deploy') {
            when { expression { onDeployBranch() } }
            steps {
                dir('infra') {
                    deleteDir()
                    git url: "${INFRA_REPO}", branch: "${INFRA_BRANCH}"
                }
                withCredentials([
                    sshUserPrivateKey(credentialsId: 'azure-vm-ssh', keyFileVariable: 'SSH_KEY'),
                    string(credentialsId: 'azure-vm-ip', variable: 'VM_IP'),
                    usernamePassword(
                        credentialsId: 'ghcr-credentials',
                        usernameVariable: 'GHCR_USER',
                        passwordVariable: 'GHCR_TOKEN'
                    )
                ]) {
                    sh '''
                        set -eu
                        # Without this the stage dies with "ansible-playbook: not
                        # found" and exit 127, which says nothing about how to fix
                        # it -- and only after the image has already been built and
                        # pushed, several minutes in.
                        if ! command -v ansible-playbook >/dev/null 2>&1; then
                            echo "ansible-playbook is not installed on this Jenkins host."
                            echo "Install it with:   sudo apt install -y ansible"
                            echo "then restart Jenkins:   sudo systemctl restart jenkins"
                            exit 1
                        fi
                        cd infra/ansible
                        # The inventory is one host, so it is written here from the IP
                        # Terraform produced rather than committed to the infra repo.
                        trap 'rm -f inventory.ini' EXIT
                        {
                            echo "[app]"
                            echo "$VM_IP ansible_user=azureuser"
                        } > inventory.ini
                        ansible-playbook deploy.yml -i inventory.ini --private-key "$SSH_KEY" -e image_name="${IMAGE_NAME}" -e image_tag="${IMAGE_TAG}" -e ghcr_user="$GHCR_USER" -e ghcr_token="$GHCR_TOKEN"
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            when { expression { onDeployBranch() } }
            steps {
                withCredentials([string(credentialsId: 'azure-vm-ip', variable: 'VM_IP')]) {
                    sh '''
                        set -eu
                        for attempt in $(seq 1 30); do
                            if curl -fsS "http://$VM_IP/api/health" | grep -q '"status":"ok"'; then
                                echo "deployment healthy: http://$VM_IP/"
                                exit 0
                            fi
                            sleep 5
                        done
                        echo "health check never returned ok"
                        exit 1
                    '''
                }
            }
        }
    }

    post {
        always {
            sh '''
                docker image prune -f --filter "until=72h" || true
                # Safety net. Nothing should be writing the shared docker config
                # any more, but a stale ghcr.io credential there breaks even
                # public pulls, so never let one outlive the build.
                rm -rf "${WORKSPACE}/.docker-ghcr" || true
                docker logout ghcr.io >/dev/null 2>&1 || true
            '''
        }
        success {
            script {
                if (onDeployBranch()) {
                    echo "Deployed ${env.IMAGE_NAME}:${env.IMAGE_TAG}"
                } else {
                    echo "Built and tested ${env.IMAGE_NAME}:${env.IMAGE_TAG}; " +
                         "NOT deployed -- this is not the deploy branch " +
                         "(BRANCH_NAME=${env.BRANCH_NAME}, GIT_BRANCH=${env.GIT_BRANCH})."
                }
            }
        }
        failure {
            echo "Build ${env.BUILD_NUMBER} failed at stage: ${env.STAGE_NAME}"
        }
    }
}
