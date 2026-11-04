// =============================================================================
// Jenkinsfile — Linux World Flask app
//
//   1. Checkout                         (checkout scm + runtime/port detection)
//   2. AI Code Validation & Security    (aiAgent claudeCode(), read-only + ci-logs/ writes)
//   3. Dockerfile Audit, Build & Deploy (hadolint + Claude audit, build, run)
//   4. API Health Check                 (curl /health + /, 5 tries x 5 s)
//   5. Final Report                     (pipeline_summary_${BUILD_NUMBER}.md)
//
// Claude Code is invoked through the Jenkins "AI Agent" plugin
// (https://plugins.jenkins.io/ai-agent/) step:  aiAgent(agent: claudeCode(), ...)
// The plugin runs Claude Code headless (stream-json) and injects the Anthropic
// key from a Jenkins credential. yoloMode stays OFF; instead we pass
// `--permission-mode dontAsk --allowedTools Read,Grep,Glob,Edit(./ci-logs/**)`
// so anything not on that list is denied automatically. Claude can read the
// repo and write its report under ci-logs/ only. It cannot run shell commands
// or modify source files.
// =============================================================================

pipeline {
    // Needs an agent with docker, curl, git and Node.js/npx (used by the plugin to run Claude Code).
    // Swap for e.g. `agent { label 'docker && claude' }` on a multi-agent setup.
    agent any

    parameters {
        string(name: 'HOST_PORT', defaultValue: '8081',
               description: 'Host port the app container is published on (8080 is used by Jenkins itself).')
        string(name: 'CLAUDE_MODEL', defaultValue: '',
               description: 'Optional Claude model alias/ID; empty = CLI default.')
        string(name: 'CLAUDE_CREDENTIAL_ID', defaultValue: 'anthropic-api-key',
               description: 'Jenkins Secret-text credential with the Anthropic API key. Empty = use the agent node\'s existing Claude login.')
        booleanParam(name: 'FAIL_ON_AI_CRITICAL', defaultValue: false,
               description: 'Fail the build when Claude reports VERDICT: FAIL (critical/high findings).')
    }

    options {
        skipDefaultCheckout(true)          // Stage 1 does an explicit `checkout scm`
        disableConcurrentBuilds()          // single test container / host port
        timestamps()
        timeout(time: 45, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    environment {
        REPORT_DIR     = '/root/Jenkins/report'
        LOG_DIR        = "${env.WORKSPACE}/ci-logs"
        IMAGE_NAME     = 'app'
        IMAGE_TAG      = "${env.BUILD_NUMBER}"
        CONTAINER_NAME = 'app-test'
        HOST_PORT      = "${params.HOST_PORT ?: '8081'}"
        // Optional: absolute path to a `claude` binary. Empty = plugin default (npx).
        CLAUDE_EXECUTABLE = ''
        // Read-only repo access + write access to ci-logs/ only (for the report files)
        CLAUDE_TOOLS   = 'Read,Grep,Glob,Edit(./ci-logs/**)'
        HADOLINT_IMAGE = 'hadolint/hadolint:v2.12.0'
        // Credential (Secret text) holding the Anthropic API key; see prerequisites.
        ANTHROPIC_CRED = "${params.CLAUDE_CREDENTIAL_ID != null ? params.CLAUDE_CREDENTIAL_ID : 'anthropic-api-key'}"
    }

    stages {

        // ---------------------------------------------------------------------
        stage('1. Checkout') {
            steps {
              catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                script {
                    // Defaults so the final report is complete even if a stage is skipped
                    env.CHECKOUT_STATUS   = 'FAILED'
                    env.VALIDATION_STATUS = 'NOT_RUN'
                    env.AI_VERDICT        = 'UNKNOWN'
                    env.DOCKER_AUDIT      = 'NOT_RUN'
                    env.BUILD_STATUS      = 'NOT_RUN'
                    env.DEPLOY_STATUS     = 'NOT_RUN'
                    env.HEALTH_STATUS     = 'NOT_RUN'
                    env.HEALTH_HTTP_CODE  = 'n/a'
                    env.ROOT_HTTP_CODE    = 'n/a'
                    env.HEALTH_ATTEMPTS   = '0'

                    def scmVars = checkout scm
                    env.GIT_COMMIT = scmVars.GIT_COMMIT ?: ''
                    env.GIT_BRANCH = scmVars.GIT_BRANCH ?: (env.BRANCH_NAME ?: 'unknown')
                    env.GIT_URL    = scmVars.GIT_URL ?: 'unknown'
                }

                sh '''#!/bin/bash
                    set -euo pipefail
                    mkdir -p "$REPORT_DIR" "$LOG_DIR"

                    echo "== Agent tool check =="
                    tools="git docker curl"
                    if [ -n "${CLAUDE_EXECUTABLE:-}" ]; then tools="$tools $CLAUDE_EXECUTABLE"; else tools="$tools node npx"; fi
                    for t in $tools; do
                        command -v "$t" >/dev/null || { echo "ERROR: '$t' not found on PATH"; exit 1; }
                    done
                    docker info --format 'Docker server {{.ServerVersion}}' \
                        || { echo "ERROR: cannot reach Docker daemon (is the jenkins user in the docker group?)"; exit 1; }
                    command -v node >/dev/null && echo "Node $(node --version)"
                    rm -f "$LOG_DIR"/ai_*.md   # never reuse a previous build's AI output

                    git log -1 --pretty=format:'%H|%an|%ad|%s' --date=iso > "$LOG_DIR/commit.txt"
                    git ls-files > "$LOG_DIR/files.txt"

                    # ---- Runtime + port auto-detection ----
                    runtime=unknown
                    [ -f requirements.txt ] && runtime=python
                    [ -f package.json ]     && runtime=node
                    [ -f go.mod ]           && runtime=go
                    [ -f pom.xml ]          && runtime=java
                    grep -qi '^flask' requirements.txt 2>/dev/null && runtime="python/flask"

                    port=""
                    [ -f Dockerfile ] && port=$(awk 'toupper($1)=="EXPOSE"{split($2,a,"/"); print a[1]; exit}' Dockerfile)
                    [ -z "$port" ] && port=$(grep -oE 'port=[0-9]+' app.py 2>/dev/null | head -1 | cut -d= -f2 || true)
                    [ -z "$port" ] && port=5000

                    health=/health
                    grep -qE "route\\(['\\"]/health" app.py 2>/dev/null || health=/

                    printf 'RUNTIME=%s\\nAPP_PORT=%s\\nHEALTH_PATH=%s\\nPIPELINE_START=%s\\n' \\
                        "$runtime" "$port" "$health" "$(date '+%Y-%m-%d %H:%M:%S %Z')" > "$LOG_DIR/detect.env"
                    cat "$LOG_DIR/detect.env"
                '''

                script {
                    for (line in readFile("${env.LOG_DIR}/detect.env").trim().split('\n')) {
                        def kv = line.split('=', 2)
                        if (kv.length == 2) { env."${kv[0]}" = kv[1] }
                    }
                    def c = readFile("${env.LOG_DIR}/commit.txt").trim().tokenize('|')
                    env.GIT_AUTHOR = c.size() > 1 ? c[1] : 'unknown'
                    env.GIT_DATE   = c.size() > 2 ? c[2] : 'unknown'
                    env.GIT_MSG    = c.size() > 3 ? c[3..-1].join('|') : ''
                    env.CHECKOUT_STATUS = 'SUCCESS'
                    echo "Detected ${env.RUNTIME} app on container port ${env.APP_PORT}; health path ${env.HEALTH_PATH}"
                }
              }
            }
        }

        // ---------------------------------------------------------------------
        stage('2. AI Code Validation & Security Audit') {
            when { expression { env.CHECKOUT_STATUS == 'SUCCESS' } }
            steps {
                // Advisory stage: an AI/API failure marks the build UNSTABLE, not FAILED.
                catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE') {
                    script {
                        runClaude('''You are a senior code reviewer running inside a CI pipeline. Review the COMPLETE
source code of the repository in the current working directory: every tracked file, including
application code, templates, dependency manifests, Dockerfile and Jenkinsfile. Ignore the ci-logs/ directory.

Report:
1. Bugs and correctness problems
2. Lint / style / maintainability issues
3. Security vulnerabilities (OWASP Top 10, injection, XSS, insecure defaults,
   debug mode, secrets in code, vulnerable or unpinned dependencies)
4. Container and deployment concerns

Rules:
- Do NOT modify any repository file. The only file you may create is ci-logs/ai_validation.md.
- Never print the value of any secret you find; give only file and line.
- Cite file:line for every finding and give a severity: CRITICAL, HIGH, MEDIUM, LOW, INFO.

Write the full report with the Write tool to ci-logs/ai_validation.md as GitHub-flavoured
Markdown in exactly this layout:
VERDICT: PASS | WARN | FAIL   (FAIL if any CRITICAL/HIGH, WARN if any MEDIUM, else PASS)
## Summary
(3-6 bullet points and a count of findings per severity)
## Findings
(a table: Severity | Category | File:Line | Issue | Recommendation)
## Recommendations
''')
                    }
                    sh '''#!/bin/bash
                        set -euo pipefail
                        SRC="$LOG_DIR/ai_validation.md"
                        OUT="$REPORT_DIR/validation_report_${BUILD_NUMBER}.md"
                        [ -s "$SRC" ] || { echo "ERROR: Claude did not produce ci-logs/ai_validation.md"; exit 1; }
                        {
                            echo "# AI Code Validation Report: Build #${BUILD_NUMBER}"
                            echo ""
                            echo "- Job: ${JOB_NAME}"
                            echo "- Commit: ${GIT_COMMIT} (${GIT_BRANCH})"
                            echo "- Generated: $(date '+%Y-%m-%d %H:%M:%S %Z')"
                            echo "- Engine: Jenkins AI Agent plugin, aiAgent(claudeCode()), --permission-mode dontAsk, tools: ${CLAUDE_TOOLS}"
                            echo ""
                            echo "---"
                            echo ""
                            cat "$SRC"
                        } > "$OUT"
                        echo "Validation report written to $OUT"
                    '''
                    script {
                        def report = readFile("${env.REPORT_DIR}/validation_report_${env.BUILD_NUMBER}.md")
                        def m = (report =~ /VERDICT:\s*\**\s*(PASS|WARN|FAIL)/)
                        env.AI_VERDICT = m.find() ? m.group(1) : 'UNKNOWN'
                        m = null   // java.util.regex.Matcher is not serializable; drop before next step
                        env.VALIDATION_STATUS = 'SUCCESS'
                        echo "Claude verdict: ${env.AI_VERDICT}"
                    }
                }
                script {
                    if (env.VALIDATION_STATUS != 'SUCCESS') {
                        env.VALIDATION_STATUS = 'FAILED'
                    } else if (params.FAIL_ON_AI_CRITICAL && env.AI_VERDICT == 'FAIL') {
                        // Recorded as FAILURE but keep going so Stage 5 still writes the report
                        catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                            error('Claude reported CRITICAL/HIGH findings and FAIL_ON_AI_CRITICAL is enabled.')
                        }
                    }
                }
            }
        }

        // ---------------------------------------------------------------------
        stage('3. Dockerfile Audit, Build & Deploy') {
            when { expression { env.CHECKOUT_STATUS == 'SUCCESS' } }
            steps {
                // ---- 3a. Dockerfile audit (advisory) ----
                catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                        sh '''#!/bin/bash
                            set -uo pipefail
                            AUDIT="$LOG_DIR/dockerfile_audit.md"
                            [ -f Dockerfile ] || { echo "ERROR: no Dockerfile at repo root"; exit 1; }

                            {
                                echo "### hadolint"
                                echo "~~~"
                                docker run --rm -i "$HADOLINT_IMAGE" < Dockerfile 2>&1 || true
                                echo "~~~"
                                echo ""
                                echo "### Static checks"
                                grep -qE '^USER[[:space:]]+[^r0]' Dockerfile && echo "- [x] Runs as non-root USER" || echo "- [ ] **No non-root USER**"
                                grep -q '^HEALTHCHECK' Dockerfile  && echo "- [x] HEALTHCHECK defined"   || echo "- [ ] No HEALTHCHECK"
                                grep -qE '^FROM[^:]+:latest|^FROM[[:space:]]+[^:]+$' Dockerfile \
                                    && echo "- [ ] **Base image not pinned (latest/untagged)**" || echo "- [x] Base image tag pinned"
                                grep -qiE '^(ENV|ARG).*(PASSWORD|SECRET|TOKEN|API_KEY)' Dockerfile \
                                    && echo "- [ ] **Possible secret in ENV/ARG**" || echo "- [x] No secrets in ENV/ARG"
                                [ -f .dockerignore ] && echo "- [x] .dockerignore present" || echo "- [ ] No .dockerignore"
                            } > "$AUDIT" 2>&1
                        '''

                        // Claude review of the Dockerfile; a failure here does not skip the build
                        catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                            script {
                                runClaude('''Audit ./Dockerfile (and .dockerignore if present) in this repository for Docker
best practices and security: base image pinning, non-root user, layer caching, image size, secrets,
HEALTHCHECK, exposed ports, production WSGI server. Do NOT modify any repository file.
Write your result with the Write tool to ci-logs/ai_dockerfile_review.md as concise Markdown:
a bullet list of findings, each with severity (HIGH/MEDIUM/LOW) and a one-line fix.''')
                            }
                        }

                        sh '''#!/bin/bash
                            set -uo pipefail
                            AUDIT="$LOG_DIR/dockerfile_audit.md"
                            {
                                echo ""
                                echo "### Claude Code Dockerfile review"
                                if [ -s "$LOG_DIR/ai_dockerfile_review.md" ]; then
                                    cat "$LOG_DIR/ai_dockerfile_review.md"
                                else
                                    echo "_Claude Dockerfile review unavailable._"
                                fi
                            } >> "$AUDIT"
                            cp "$AUDIT" "$REPORT_DIR/dockerfile_audit_${BUILD_NUMBER}.md"
                            cat "$AUDIT"
                        '''
                    script { env.DOCKER_AUDIT = 'DONE' }
                }

                // ---- 3b. Build + 3c. Deploy (fatal on error) ----
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                    sh '''#!/bin/bash
                        set -euo pipefail
                        echo "== docker build ${IMAGE_NAME}:${IMAGE_TAG} =="
                        docker build \
                            --pull \
                            --label "org.opencontainers.image.revision=${GIT_COMMIT}" \
                            --label "jenkins.build=${BUILD_NUMBER}" \
                            -t "${IMAGE_NAME}:${IMAGE_TAG}" \
                            -t "${IMAGE_NAME}:latest" \
                            . 2>&1 | tee "$LOG_DIR/docker_build.log"
                    '''
                    script { env.BUILD_STATUS = 'SUCCESS' }

                    sh '''#!/bin/bash
                        set -euo pipefail
                        {
                            echo "== Removing previous container (if any) =="
                            if docker ps -a --format '{{.Names}}' | grep -qx "$CONTAINER_NAME"; then
                                docker stop "$CONTAINER_NAME" || true
                                docker rm -f "$CONTAINER_NAME"
                            else
                                echo "No previous '$CONTAINER_NAME' container."
                            fi

                            echo "== Starting ${IMAGE_NAME}:${IMAGE_TAG} on host :${HOST_PORT} -> container :${APP_PORT} =="
                            docker run -d \
                                --name "$CONTAINER_NAME" \
                                -p "${HOST_PORT}:${APP_PORT}" \
                                --restart unless-stopped \
                                --read-only --tmpfs /tmp \
                                --cap-drop ALL \
                                --security-opt no-new-privileges \
                                --memory 256m --cpus 0.5 \
                                --label "jenkins.build=${BUILD_NUMBER}" \
                                "${IMAGE_NAME}:${IMAGE_TAG}"

                            docker ps --filter "name=^/${CONTAINER_NAME}$" \
                                --format 'table {{.ID}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}'
                        } 2>&1 | tee "$LOG_DIR/docker_deploy.log"
                    '''
                    script { env.DEPLOY_STATUS = 'SUCCESS' }
                }
                script {
                    if (env.BUILD_STATUS  != 'SUCCESS') { env.BUILD_STATUS  = 'FAILED' }
                    if (env.DEPLOY_STATUS != 'SUCCESS') {
                        env.DEPLOY_STATUS = (env.BUILD_STATUS == 'SUCCESS') ? 'FAILED' : 'SKIPPED'
                    }
                }
            }
        }

        // ---------------------------------------------------------------------
        stage('4. API Health Check') {
            when { expression { env.DEPLOY_STATUS == 'SUCCESS' } }
            steps {
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                    sh '''#!/bin/bash
                        set -uo pipefail
                        URL="http://localhost:${HOST_PORT}${HEALTH_PATH}"
                        ROOT_URL="http://localhost:${HOST_PORT}/"
                        MAX=5; SLEEP=5; code=000; i=0
                        : > "$LOG_DIR/health.log"

                        for i in $(seq 1 $MAX); do
                            code=$(curl -s -o "$LOG_DIR/health_body.txt" -w '%{http_code}' \
                                        --connect-timeout 3 --max-time 5 "$URL" || true)
                            echo "Attempt $i/$MAX: GET $URL -> HTTP $code" | tee -a "$LOG_DIR/health.log"
                            [ "$code" = "200" ] && break
                            [ "$i" -lt "$MAX" ] && sleep "$SLEEP"
                        done

                        root_code=$(curl -s -o /dev/null -w '%{http_code}' --max-time 5 "$ROOT_URL" || true)
                        echo "GET $ROOT_URL -> HTTP $root_code" | tee -a "$LOG_DIR/health.log"
                        echo "Body: $(head -c 500 "$LOG_DIR/health_body.txt" 2>/dev/null)" | tee -a "$LOG_DIR/health.log"

                        printf 'HEALTH_HTTP_CODE=%s\\nROOT_HTTP_CODE=%s\\nHEALTH_ATTEMPTS=%s\\n' \
                            "$code" "$root_code" "$i" > "$LOG_DIR/health.env"

                        if [ "$code" != "200" ]; then
                            echo "Service NOT healthy; container logs:"
                            docker logs --tail 50 "$CONTAINER_NAME" 2>&1 | tee -a "$LOG_DIR/health.log"
                            exit 1
                        fi
                        echo "Service healthy (HTTP 200)."
                    '''
                }
                script {
                    if (fileExists("${env.LOG_DIR}/health.env")) {
                        for (line in readFile("${env.LOG_DIR}/health.env").trim().split('\n')) {
                            def kv = line.split('=', 2)
                            if (kv.length == 2) { env."${kv[0]}" = kv[1] }
                        }
                    }
                    env.HEALTH_STATUS = (env.HEALTH_HTTP_CODE == '200') ? 'HEALTHY' : 'UNHEALTHY'
                }
            }
        }

        // ---------------------------------------------------------------------
        stage('5. Final Report') {
            steps {
                script {
                    if (env.DEPLOY_STATUS != 'SUCCESS' && env.HEALTH_STATUS == 'NOT_RUN') {
                        env.HEALTH_STATUS = 'SKIPPED'
                    }
                    env.RESULT_SO_FAR = currentBuild.currentResult
                }
                sh '''#!/bin/bash
                    set -uo pipefail
                    OUT="$REPORT_DIR/pipeline_summary_${BUILD_NUMBER}.md"
                    VAL="$REPORT_DIR/validation_report_${BUILD_NUMBER}.md"
                    mkdir -p "$REPORT_DIR"

                    section_tail () { # $1=file $2=lines
                        echo "~~~"
                        if [ -s "$1" ]; then tail -n "$2" "$1"; else echo "(no output captured)"; fi
                        echo "~~~"
                    }

                    {
                        echo "# Pipeline Summary: ${JOB_NAME} #${BUILD_NUMBER}"
                        echo ""
                        echo "| Item | Value |"
                        echo "|---|---|"
                        echo "| Overall result (at report time) | **${RESULT_SO_FAR}** |"
                        echo "| Started | ${PIPELINE_START} |"
                        echo "| Finished | $(date '+%Y-%m-%d %H:%M:%S %Z') |"
                        echo "| Build URL | ${BUILD_URL:-n/a} |"
                        echo "| Workspace | ${WORKSPACE} |"
                        echo "| Agent | ${NODE_NAME:-unknown} |"
                        echo ""
                        echo "## Stage Status"
                        echo ""
                        echo "| # | Stage | Status |"
                        echo "|---|---|---|"
                        echo "| 1 | Checkout | ${CHECKOUT_STATUS} |"
                        echo "| 2 | AI Code Validation & Security Audit | ${VALIDATION_STATUS} (verdict: ${AI_VERDICT}) |"
                        echo "| 3a | Dockerfile Audit | ${DOCKER_AUDIT} |"
                        echo "| 3b | Docker Build | ${BUILD_STATUS} |"
                        echo "| 3c | Container Deploy | ${DEPLOY_STATUS} |"
                        echo "| 4 | API Health Check | ${HEALTH_STATUS} |"
                        echo ""
                        echo "## 1. Checkout"
                        echo ""
                        echo "- Repository: ${GIT_URL:-unknown}"
                        echo "- Branch: ${GIT_BRANCH:-unknown}"
                        echo "- Commit: ${GIT_COMMIT:-unknown}"
                        echo "- Author: ${GIT_AUTHOR:-unknown}"
                        echo "- Date: ${GIT_DATE:-unknown}"
                        echo "- Message: ${GIT_MSG:-}"
                        echo "- Detected runtime: ${RUNTIME:-unknown}; container port ${APP_PORT:-?}; health path ${HEALTH_PATH:-?}"
                        echo "- Tracked files: $(wc -l < "$LOG_DIR/files.txt" 2>/dev/null || echo '?')"
                        echo ""
                        echo "## 2. Claude Code Validation"
                        echo ""
                        echo "Full report: $VAL"
                        echo ""
                        if [ -s "$VAL" ]; then
                            grep -m1 -E 'VERDICT:' "$VAL" || true
                            echo ""
                            # Copy the "## Summary" section only
                            awk '/^## Summary/{f=1} /^## /&&!/^## Summary/{if(f)exit} f' "$VAL"
                        else
                            echo "_Validation report not available._"
                        fi
                        echo ""
                        echo "## 3. Docker"
                        echo ""
                        echo "- Image: ${IMAGE_NAME}:${IMAGE_TAG}"
                        echo "- Container: ${CONTAINER_NAME} (host :${HOST_PORT} -> container :${APP_PORT:-?})"
                        echo "- Image ID: $(docker image inspect --format '{{.Id}}' "${IMAGE_NAME}:${IMAGE_TAG}" 2>/dev/null || echo n/a)"
                        echo "- Image size: $(docker image inspect --format '{{.Size}}' "${IMAGE_NAME}:${IMAGE_TAG}" 2>/dev/null | awk '{printf "%.1f MB", $1/1048576}' || echo n/a)"
                        echo ""
                        echo "### Dockerfile audit"
                        echo ""
                        if [ -s "$LOG_DIR/dockerfile_audit.md" ]; then cat "$LOG_DIR/dockerfile_audit.md"; else echo "_Not available._"; fi
                        echo ""
                        echo "### Build log (last 40 lines)"
                        section_tail "$LOG_DIR/docker_build.log" 40
                        echo ""
                        echo "### Deploy log"
                        section_tail "$LOG_DIR/docker_deploy.log" 40
                        echo ""
                        echo "## 4. API Health Check"
                        echo ""
                        echo "- Endpoint: http://localhost:${HOST_PORT}${HEALTH_PATH:-/health}"
                        echo "- Status: **${HEALTH_STATUS}**"
                        echo "- Health HTTP code: ${HEALTH_HTTP_CODE} after ${HEALTH_ATTEMPTS} attempt(s) (max 5, 5 s apart)"
                        echo "- Root (/) HTTP code: ${ROOT_HTTP_CODE}"
                        echo ""
                        section_tail "$LOG_DIR/health.log" 60
                    } > "$OUT"

                    echo "Pipeline summary written to $OUT"
                    cat "$OUT"
                '''
            }
        }
    }

    post {
        always {
            script {
                // Copy reports into the workspace so they are also archived with the build
                sh '''#!/bin/bash
                    mkdir -p "$LOG_DIR/report"
                    cp -f "$REPORT_DIR"/*_"${BUILD_NUMBER}".md "$LOG_DIR/report/" 2>/dev/null || true
                '''
                archiveArtifacts artifacts: 'ci-logs/**', allowEmptyArchive: true, fingerprint: false
            }
        }
        success  { echo "Build #${env.BUILD_NUMBER} OK: app is live at http://localhost:${env.HOST_PORT}" }
        unstable { echo "Build #${env.BUILD_NUMBER} UNSTABLE: see ${env.REPORT_DIR}/pipeline_summary_${env.BUILD_NUMBER}.md" }
        failure  { echo "Build #${env.BUILD_NUMBER} FAILED: see ${env.REPORT_DIR}/pipeline_summary_${env.BUILD_NUMBER}.md" }
    }
}

// -----------------------------------------------------------------------------
// Runs Claude Code through the Jenkins AI Agent plugin (aiAgent step).
//  - apiCredentialsId: Secret-text credential, injected as ANTHROPIC_API_KEY
//  - yoloMode false + dontAsk + allowlist = read repo, write only ci-logs/**
//  - failOnAgentError: non-zero exit raises an error (caught by the caller)
// -----------------------------------------------------------------------------
def runClaude(String prompt) {
    def cfg = [
        agent           : claudeCode(),
        prompt          : prompt,
        yoloMode        : false,
        failOnAgentError: true,
        extraArgs       : "--permission-mode dontAsk --allowedTools ${env.CLAUDE_TOOLS}"
    ]
    if (env.ANTHROPIC_CRED?.trim())      { cfg.apiCredentialsId = env.ANTHROPIC_CRED.trim() }
    if (params.CLAUDE_MODEL?.trim())     { cfg.model          = params.CLAUDE_MODEL.trim() }
    if (env.CLAUDE_EXECUTABLE?.trim())   { cfg.executablePath = env.CLAUDE_EXECUTABLE.trim() }
    timeout(time: 20, unit: 'MINUTES') {
        aiAgent(cfg)
    }
}
