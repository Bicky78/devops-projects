# Jenkins Scenario-Based & Day-to-Day Interview Questions

Real-world troubleshooting scenarios, day-to-day operational tasks, and practical problem-solving questions for Jenkins interviews — focused on what DevOps engineers actually face in production.

---

## Table of Contents

- [Troubleshooting Scenarios](#troubleshooting-scenarios)
- [Pipeline & Build Scenarios](#pipeline--build-scenarios)
- [Day-to-Day Operations Questions](#day-to-day-operations-questions)
- [Scaling & Performance Scenarios](#scaling--performance-scenarios)
- [Security & Credential Scenarios](#security--credential-scenarios)
- [Deployment & Release Scenarios](#deployment--release-scenarios)
- [Migration & Administration Scenarios](#migration--administration-scenarios)

---

## Troubleshooting Scenarios

### 1. Jenkins is integrated with Git, but a new commit doesn't trigger a build. How do you troubleshoot?

**Answer:**

```bash
# Systematic troubleshooting:
```

| Check | How |
|-------|-----|
| **Webhook config** | Verify webhook URL points to correct Jenkins server and job (e.g., `https://jenkins.example.com/github-webhook/`) |
| **Webhook delivery** | Check Git provider's webhook delivery logs (GitHub → Settings → Webhooks → Recent Deliveries) |
| **Job trigger config** | Verify job is configured for "GitHub hook trigger" or correct trigger type |
| **SCM Polling** | If using Poll SCM, verify schedule is set and correct |
| **Git Plugin version** | Update Git plugin — old versions have known bugs |
| **Jenkins logs** | Check `Manage Jenkins → System Log` for webhook/SCM errors |
| **Network/Firewall** | Ensure Git provider can reach Jenkins (not blocked by firewall/security groups) |
| **Branch filter** | Verify branch specifier matches the pushed branch (e.g., `*/main` vs `*/master`) |

```bash
# Quick test — trigger build via webhook manually
curl -X POST https://jenkins.example.com/github-webhook/
```

---

### 2. A Jenkins build is failing due to a lack of available nodes. How would you resolve this?

**Answer:**

```bash
# 1. Check node status
Manage Jenkins → Manage Nodes and Clouds
# Are agents online? Any marked offline or disconnected?

# 2. Check executor availability
# Look at build queue — how many jobs waiting?

# 3. Check resource allocation on agents
# SSH to agents and check CPU/memory/disk
```

**Immediate fixes:**
- Reconnect offline agents
- Increase executors on existing agents
- Restart unresponsive agent services

**Long-term fixes:**

| Solution | When to Use |
|----------|-------------|
| Add more agents | Consistent high demand |
| Use cloud agents (EC2/K8s) | Variable demand — auto-scale |
| Optimize build times | Reduce queue pressure |
| Review label assignments | Some labels may have too few agents |
| Set build timeouts | Prevent stuck builds holding executors |
| Enable concurrent builds | If jobs are safe to parallelize |

---

### 3. A Jenkins pipeline job is producing inconsistent results. How would you troubleshoot?

**Answer:**

**Step-by-step approach:**

1. **Check job logs** — Compare logs from passing and failing runs
2. **Check for environment differences:**
   ```bash
   # Add to pipeline to expose environment
   stage('Debug') {
       steps {
           sh 'env | sort'
           sh 'java -version'
           sh 'mvn --version'
           sh 'df -h'
           sh 'free -m'
       }
   }
   ```

3. **Check agent consistency** — Does failure correlate with specific agents?
   ```groovy
   // Add to see which agent ran the build
   echo "Running on: ${env.NODE_NAME}"
   ```

4. **Check for workspace contamination** — Old files from previous builds
   ```groovy
   // Add workspace cleanup
   post { always { cleanWs() } }
   ```

5. **Check external dependencies** — Flaky tests, network timeouts, unreliable services
6. **Check plugin versions** — Recent updates may have introduced bugs
7. **Check for race conditions** — If parallel stages share resources

---

### 4. A build is stuck and appears to hang indefinitely. What do you do?

**Answer:**

```bash
# 1. Check console output — is it waiting for input?
# Manual input step may be blocking: input "Proceed with deployment?"

# 2. Check if it's waiting for an executor
# Build may be queued waiting for a specific label

# 3. Check for deadlocks in the agent
# SSH to agent, check process tree: ps aux | grep java

# 4. Set a timeout to prevent this in the future:
```

```groovy
pipeline {
    agent any
    options {
        timeout(time: 30, unit: 'MINUTES')  // Global timeout
    }
    stages {
        stage('Deploy') {
            options {
                timeout(time: 10, unit: 'MINUTES')  // Stage timeout
            }
            steps {
                sh 'deploy.sh'
            }
        }
    }
}
```

**Immediate resolution:** Kill the build via Jenkins UI → Build → Stop or abort from CLI.

---

### 5. Jenkins is not sending notifications after a job fails. How do you troubleshoot?

**Answer:**

```bash
# 1. Check post-build actions
# Verify notification is configured in the correct post {} condition
# Common mistake: notification in success {} but not failure {}

# 2. Check notification plugin configuration
# Manage Jenkins → Configure System → Email / Slack settings

# 3. Test notification manually
# Email: Manage Jenkins → Configure System → E-mail Notification → Test
# Slack: Send a test message from Slack config section

# 4. Check Jenkins system logs
# Manage Jenkins → System Log → search for email/slack errors

# 5. Check network connectivity
# Can Jenkins reach SMTP server / Slack API?

# 6. Check credentials
# Verify SMTP password / Slack token is valid and not expired
```

**Correct notification pattern:**
```groovy
post {
    failure {
        mail to: 'team@example.com',
             subject: "FAILED: ${currentBuild.fullDisplayName}",
             body: "Check: ${BUILD_URL}"
    }
    success {
        slackSend channel: '#ci-cd', message: "Build passed!"
    }
    always {
        // This runs regardless — good for cleanup
        junit '**/test-results/*.xml'
    }
}
```

---

### 6. Tests pass locally but fail in Jenkins. How do you debug?

**Answer:**

**Common causes:**

| Cause | Investigation |
|-------|--------------|
| **Different tool versions** | Compare Java, Maven, Node versions: `java -version`, `mvn --version` |
| **Missing env variables** | Check `env | sort` on agent vs local |
| **Timezone differences** | Agent may be UTC while local is different |
| **File system differences** | Linux agent vs macOS local (case sensitivity, path separators) |
| **Network access** | Agent may not reach external services (firewall, proxy) |
| **Workspace state** | Previous builds left stale files; add `cleanWs()` |
| **Memory limits** | Agent has less memory; tests OOM |
| **Parallel execution** | Tests assume sequential execution but run in parallel |

```groovy
// Debug pipeline — add environment dump
stage('Debug') {
    steps {
        sh 'echo "Node: ${NODE_NAME}"'
        sh 'env | sort'
        sh 'pwd && ls -la'
        sh 'java -version 2>&1'
        sh 'cat /etc/os-release'
    }
}
```

---

### 7. Jenkins job fails to run due to a missing dependency in the build environment. How do you fix?

**Answer:**

```bash
# 1. Check Global Tool Configuration
# Manage Jenkins → Global Tool Configuration
# Ensure Maven, JDK, Node.js, etc. are defined with correct versions

# 2. Install missing tools on agent
# SSH to agent and install manually or via user data script

# 3. Use Docker agent for isolated environments
```

```groovy
// Option A: Use tools directive
pipeline {
    agent any
    tools {
        maven 'Maven-3.8'
        jdk 'JDK-17'
    }
    stages { ... }
}

// Option B: Use Docker agent (guaranteed dependencies)
pipeline {
    agent {
        docker { image 'maven:3.8-openjdk-17' }
    }
    stages {
        stage('Build') {
            steps { sh 'mvn clean package' }
        }
    }
}

// Option C: Install inline (last resort)
stage('Setup') {
    steps {
        sh 'npm install'           // Node.js
        sh 'pip install -r requirements.txt'  // Python
    }
}
```

---

### 8. Agent frequently disconnects from the controller. What steps would you take?

**Answer:**

| Check | Solution |
|-------|----------|
| **Network stability** | Check for packet loss, firewall rules, security groups |
| **Agent logs** | Review agent's `remoting.jar` logs on the agent machine |
| **Java version mismatch** | Ensure same major Java version on controller and agent |
| **Resource exhaustion** | Check CPU/memory/disk on agent — OOM can kill agent process |
| **Ping thread timeout** | Increase in Manage Jenkins → Configure Global Security → Agent → TCP Port |
| **Keepalive settings** | Increase TCP keepalive and connection timeout |
| **Agent process management** | Run agent as systemd service with `Restart=always` |
| **Network MTU issues** | Check for MTU mismatches in cloud/VPN environments |

```ini
# /etc/systemd/system/jenkins-agent.service
[Service]
Restart=always
RestartSec=10
ExecStart=/usr/bin/java -jar /opt/jenkins/agent.jar \
  -url https://jenkins.example.com \
  -secret <secret> \
  -name agent-1 \
  -workDir /var/jenkins-workspace
```

---

## Pipeline & Build Scenarios

### 9. You want to trigger a Jenkins build every time a change is made to a specific Git branch. How do you configure?

**Answer:**

**Method 1: Webhook (recommended)**
```groovy
pipeline {
    agent any
    triggers {
        githubPush()  // Requires GitHub plugin + webhook configured
    }
    stages {
        stage('Build') {
            when {
                branch 'main'  // Only build on main branch
            }
            steps {
                sh 'mvn clean package'
            }
        }
    }
}
```

**Method 2: SCM Polling**
```groovy
pipeline {
    agent any
    triggers {
        pollSCM('H/5 * * * *')  // Check every 5 minutes
    }
    // ...
}
```

In the job configuration, set Branch Specifier to `*/main` to only build the main branch.

**Method 3: Multibranch Pipeline**
- Automatically creates jobs for each branch with a Jenkinsfile
- Most scalable for teams with many branches

---

### 10. How do you handle a Jenkins job that takes too long due to external dependencies?

**Answer:**

```groovy
pipeline {
    agent any
    options {
        timeout(time: 45, unit: 'MINUTES')  // Global timeout
    }
    stages {
        stage('External Call') {
            options {
                timeout(time: 10, unit: 'MINUTES')
                retry(3)  // Retry up to 3 times on failure
            }
            steps {
                sh '''
                    curl --retry 3 --retry-delay 5 --max-time 30 \
                         https://external-service.example.com/api
                '''
            }
        }

        stage('Parallel External Tasks') {
            parallel {
                stage('API Test') {
                    steps { sh 'run-api-tests.sh' }
                }
                stage('Integration Test') {
                    steps { sh 'run-integration-tests.sh' }
                }
            }
        }
    }
}
```

**Strategies:**
- **Timeout + retry** for intermittent external failures
- **Parallelize** independent external calls
- **Cache dependencies** (Maven `.m2`, npm `node_modules`)
- **Mock external services** in tests where possible
- **Move slow tasks to async** — trigger and poll for status

---

### 11. A production deployment failed even though the build passed. How would you prevent this in the future?

**Answer:**

```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps { sh 'mvn clean package' }
        }

        // Quality gates BEFORE deployment
        stage('Quality Gates') {
            parallel {
                stage('Unit Tests') {
                    steps { sh 'mvn test' }
                }
                stage('Code Coverage') {
                    steps {
                        sh 'mvn jacoco:report'
                        jacoco minimumLineCoverage: '80'
                    }
                }
                stage('Static Analysis') {
                    steps {
                        withSonarQubeEnv('sonarqube') {
                            sh 'mvn sonar:sonar'
                        }
                        waitForQualityGate abortPipeline: true
                    }
                }
                stage('Security Scan') {
                    steps { sh 'trivy image myapp:${BUILD_NUMBER}' }
                }
            }
        }

        // Staging deployment + smoke tests
        stage('Deploy Staging') {
            steps { sh 'deploy.sh --env staging' }
        }
        stage('Smoke Test') {
            steps { sh 'smoke-test.sh --env staging' }
        }

        // Manual approval gate
        stage('Production Approval') {
            input {
                message "Deploy to production?"
                submitter "release-managers"
            }
            steps { sh 'deploy.sh --env production' }
        }
    }
}
```

---

### 12. Jenkins plugins were updated and several pipelines stopped working. How do you handle it?

**Answer:**

**Immediate response:**
```bash
# 1. Identify broken plugins — check Jenkins system log
Manage Jenkins → System Log

# 2. Roll back incompatible plugins
Manage Jenkins → Manage Plugins → Installed → Downgrade
# Or manually replace plugin .jpi files in $JENKINS_HOME/plugins/

# 3. Restart Jenkins after rollback
<jenkins-url>/safeRestart
```

**Prevention:**
```bash
# 1. Maintain a staging Jenkins instance
# Test plugin updates there before production

# 2. Pin plugin versions in version control
# plugins.txt (for Docker-based Jenkins)
git:4.14.0
pipeline-model-definition:2.2131
docker-workflow:1.29

# 3. Document plugin dependencies
# Use "Plugin Usage" plugin to see which jobs use which plugins

# 4. Never update all plugins at once
# Update in small batches, test after each batch

# 5. Schedule plugin updates during maintenance windows
```

---

### 13. Different teams use different Jenkinsfile styles, causing confusion. How do you improve maintainability?

**Answer:**

1. **Standardize on Declarative syntax** — easier to read and validate
2. **Create Shared Libraries** with common pipeline templates:

```groovy
// vars/standardPipeline.groovy (Shared Library)
def call(Map config) {
    pipeline {
        agent { label config.agent ?: 'any' }
        stages {
            stage('Checkout') { steps { checkout scm } }
            stage('Build') { steps { sh config.buildCmd ?: 'mvn clean package' } }
            stage('Test') { steps { sh config.testCmd ?: 'mvn test' } }
            stage('Deploy') {
                when { branch 'main' }
                steps { sh config.deployCmd ?: 'echo No deploy configured' }
            }
        }
        post {
            failure { slackSend message: "Build Failed: ${currentBuild.fullDisplayName}" }
        }
    }
}
```

```groovy
// Team's Jenkinsfile — clean and simple
@Library('company-pipeline-lib') _
standardPipeline(
    agent: 'maven',
    buildCmd: 'mvn clean package',
    testCmd: 'mvn test',
    deployCmd: 'deploy.sh --env staging'
)
```

3. **Document pipeline standards** — README in the shared library repo
4. **Enforce via code review** — Jenkinsfile changes require approval
5. **Provide starter templates** — For new projects

---

## Day-to-Day Operations Questions

### 14. How do you check the status of all Jenkins agents?

**Answer:**

```bash
# Via UI
Manage Jenkins → Manage Nodes and Clouds
# Shows: Online/Offline status, executors, labels, free disk space

# Via Jenkins CLI
java -jar jenkins-cli.jar -s http://localhost:8080/ list-nodes

# Via API (JSON)
curl -s http://localhost:8080/computer/api/json?pretty=true

# Via Groovy Script Console (Manage Jenkins → Script Console)
Jenkins.instance.computers.each { computer ->
    println "${computer.name}: ${computer.isOnline() ? 'ONLINE' : 'OFFLINE'} " +
            "- Executors: ${computer.numExecutors} " +
            "- Idle: ${computer.countIdle()}"
}
```

---

### 15. How do you view and manage the build queue?

**Answer:**

```bash
# Via UI
# Dashboard → Build Queue (left sidebar)

# Via API
curl -s http://localhost:8080/queue/api/json?pretty=true

# Cancel all queued builds (Groovy Console)
Jenkins.instance.queue.items.each {
    Jenkins.instance.queue.cancel(it.task)
}

# Cancel specific job from queue
Jenkins.instance.queue.items.findAll {
    it.task.name == 'my-job'
}.each {
    Jenkins.instance.queue.cancel(it.task)
}
```

---

### 16. How do you clean up old builds and disk space?

**Answer:**

```groovy
// Method 1: Set retention in Jenkinsfile
pipeline {
    agent any
    options {
        buildDiscarder(logRotator(
            numToKeepStr: '10',        // Keep last 10 builds
            daysToKeepStr: '30',       // Or builds from last 30 days
            artifactNumToKeepStr: '5'  // Keep artifacts from last 5 builds
        ))
    }
    stages { ... }
}
```

```bash
# Method 2: Groovy Script Console — bulk cleanup
Jenkins.instance.allItems(hudson.model.Job).each { job ->
    job.builds.findAll { build ->
        build.number < (job.lastBuild.number - 20)  // Keep last 20
    }.each { build ->
        build.delete()
    }
}

# Method 3: CLI cleanup
# Clean workspaces for all offline agents
Manage Jenkins → Manage Nodes → Select Node → Disconnect → Clean

# Method 4: Workspace Cleanup plugin
# Auto-cleans workspace after build via post { always { cleanWs() } }
```

---

### 17. How do you manage Jenkins jobs at scale (hundreds of jobs)?

**Answer:**

| Approach | Tool/Plugin |
|----------|------------|
| **Generate jobs from code** | Job DSL plugin or Jenkins Job Builder |
| **Organize into folders** | Folders plugin — logical groups with own permissions |
| **Shared pipeline templates** | Shared Libraries — write once, use everywhere |
| **Naming convention** | `project-environment-action` (e.g., `webapp-prod-deploy`) |
| **Auto-discovery** | Multibranch Pipeline / Organization Folder — scan repos |
| **Audit unused jobs** | Identify jobs with no recent builds — archive or delete |
| **Configuration as Code** | JCasC — system config in version control |

```groovy
// Job DSL example — generate 10 similar jobs
['service-a', 'service-b', 'service-c'].each { service ->
    pipelineJob("${service}-build") {
        definition {
            cpsScm {
                scm {
                    git {
                        remote { url("https://github.com/org/${service}.git") }
                    }
                }
                scriptPath('Jenkinsfile')
            }
        }
    }
}
```

---

### 18. How do you update Jenkins and its plugins safely?

**Answer:**

```bash
# 1. BACKUP first
# ThinBackup plugin or manual: cp -r $JENKINS_HOME /backup/jenkins-$(date +%Y%m%d)

# 2. Check release notes for breaking changes
# https://www.jenkins.io/changelog/

# 3. Test in staging
# Maintain a staging Jenkins instance with same plugins/config
# Apply updates there first and verify pipelines

# 4. Update Jenkins
sudo systemctl stop jenkins
sudo yum update jenkins    # or apt-get upgrade jenkins
sudo systemctl start jenkins

# 5. Update plugins in small batches
# Manage Jenkins → Manage Plugins → Updates
# Select 5-10 at a time, restart, test

# 6. Verify after update
# Run critical pipelines
# Check System Log for errors
# Verify agent connectivity
```

---

### 19. How do you view and export a job's configuration?

**Answer:**

```bash
# View job config via API (XML)
curl -s http://localhost:8080/job/<job-name>/config.xml

# Download and save
curl -s http://localhost:8080/job/<job-name>/config.xml > job-config.xml

# Import job config
curl -X POST http://localhost:8080/createItem?name=<new-job-name> \
  --header "Content-Type: application/xml" \
  --data-binary @job-config.xml

# Export all jobs (Groovy Console)
Jenkins.instance.allItems(hudson.model.Job).each { job ->
    println "${job.fullName}: ${job.configFile.file.absolutePath}"
}
```

---

### 20. How do you check why a specific build failed?

**Answer:**

```bash
# 1. Console Output — primary source of truth
# Job → Build # → Console Output (or Pipeline Steps for pipeline jobs)

# 2. Blue Ocean view — visual stage-by-stage
# Job → Open in Blue Ocean → Click failed stage

# 3. Build log via API
curl -s http://localhost:8080/job/<job>/lastBuild/consoleText

# 4. Check build cause
# "Started by user", "Started by SCM change", "Started by upstream project"

# 5. Check environment variables
# Job → Build # → Environment Variables (link in sidebar)

# 6. Check changes (SCM diff)
# Job → Build # → Changes (shows commits since last build)

# 7. Check test results
# Job → Build # → Test Result (if JUnit plugin configured)
```

---

## Scaling & Performance Scenarios

### 21. Your Jenkins master is running out of resources. How would you scale?

**Answer:**

**Immediate relief:**
```bash
# 1. Disable executors on controller
Manage Jenkins → Configure System → # of executors → 0

# 2. Clean up disk space
# Remove old builds, workspaces, unused plugins
find $JENKINS_HOME/workspace -maxdepth 1 -mtime +30 -exec rm -rf {} +

# 3. Increase JVM heap
# Edit /etc/default/jenkins or systemd unit
JAVA_OPTS="-Xms2g -Xmx4g -XX:+UseG1GC"
```

**Long-term scaling:**

| Strategy | Implementation |
|----------|---------------|
| **Add static agents** | Dedicated VMs with specific labels |
| **Dynamic cloud agents** | EC2 Plugin, Kubernetes Plugin, Docker Plugin |
| **Offload artifacts** | Push to Nexus/Artifactory instead of Jenkins |
| **Set retention policies** | `buildDiscarder` on every job |
| **Optimize pipelines** | Parallel stages, caching, shallow clone |
| **Separate concerns** | Different controllers per team/environment |

---

### 22. Build times have increased noticeably over the past few weeks. What do you investigate?

**Answer:**

```bash
# 1. Compare recent build logs vs older ones — which stages got slower?
# Use timestamps() plugin to see per-step timing

# 2. Check checkout times
# Slow checkout = repo grew (large binaries committed?)
# Fix: shallow clone
```

```groovy
checkout([$class: 'GitSCM',
    extensions: [[$class: 'CloneOption', depth: 1, shallow: true]],
    ...
])
```

```bash
# 3. Check dependency download times
# Maven/npm downloading everything from scratch?
# Fix: Cache dependencies

# 4. Check test suite growth
# New tests added without optimization?
# Fix: Parallel test execution

# 5. Check Jenkinsfile changes
# New stages added? Parallelism removed?

# 6. Check agent health
# Disk full causing I/O slowdown?
# Fix: Clean workspace, prune Docker images

# 7. Check build queue wait time vs execution time
# Long queue = need more agents
# Long execution = optimize pipeline
```

---

### 23. Builds are becoming slow due to dependency downloads. How do you optimize?

**Answer:**

```groovy
// Maven — cache .m2 repository
pipeline {
    agent {
        docker {
            image 'maven:3.8-openjdk-17'
            args '-v $HOME/.m2:/root/.m2'  // Persist Maven cache
        }
    }
    stages {
        stage('Build') {
            steps { sh 'mvn clean package -o' }  // -o = offline if cache exists
        }
    }
}

// Node.js — cache node_modules
pipeline {
    agent any
    stages {
        stage('Install') {
            steps {
                sh 'npm ci --cache .npm'  // Uses lock file, caches packages
            }
        }
    }
}

// Docker — use BuildKit cache
stage('Docker Build') {
    steps {
        sh '''
            DOCKER_BUILDKIT=1 docker build \
                --cache-from registry.example.com/myapp:latest \
                -t myapp:${BUILD_NUMBER} .
        '''
    }
}
```

**Other optimizations:**
- Use **Nexus/Artifactory** as a proxy repository (mirrors Maven Central)
- Enable **shallow git clone** (`--depth 1`)
- Use **stash/unstash** instead of re-building between stages
- Configure agent with **local Docker registry mirror**

---

## Security & Credential Scenarios

### 24. Sensitive credentials are hardcoded in Jenkins pipelines. How do you secure them?

**Answer:**

**Step 1: Move all secrets to Jenkins Credentials Manager**
```bash
Manage Jenkins → Manage Credentials → Global → Add Credentials
```

**Step 2: Use `withCredentials` block in pipelines**
```groovy
// WRONG — never do this
sh 'curl -H "Authorization: Bearer abc123secret" https://api.example.com'

// CORRECT
withCredentials([string(credentialsId: 'api-token', variable: 'TOKEN')]) {
    sh 'curl -H "Authorization: Bearer $TOKEN" https://api.example.com'
}
// Jenkins automatically masks $TOKEN in console output
```

**Step 3: Restrict credential scope**
- Use **Folder-scoped credentials** — only accessible within that folder
- Use **Domain credentials** — bound to specific URLs
- Use RBAC to control who can view/edit credentials

**Step 4: For stronger security, use HashiCorp Vault**
```groovy
// Vault plugin — fetch short-lived credentials at runtime
withVault(vaultSecrets: [[path: 'secret/myapp',
                          secretValues: [[vaultKey: 'db_password', envVar: 'DB_PASS']]]]) {
    sh 'connect-to-db.sh --password $DB_PASS'
}
```

---

### 25. Credentials were leaked through a Jenkins pipeline. What steps do you take?

**Answer:**

```bash
# IMMEDIATE (in order):

# 1. Revoke and rotate the exposed credential at the SOURCE
# (GitHub token, AWS key, DB password — change it NOW)
# This limits the window of exposure

# 2. Identify scope of the leak
# - Which build exposed it?
# - What systems does the credential access?
# - Were there any unauthorized accesses? Check audit logs

# 3. Remove from Jenkins and replace
# Delete old credential, create new one with new value

# 4. Find the root cause
# Usually: shell command printed secret to output
# Or: secret passed as command-line argument (visible in ps)

# 5. Fix the pipeline
```

```groovy
// BAD — secret appears in console output
sh "deploy.sh --token ${env.SECRET_TOKEN}"

// GOOD — masked by withCredentials
withCredentials([string(credentialsId: 'deploy-token', variable: 'TOKEN')]) {
    sh 'deploy.sh --token $TOKEN'  // Single quotes prevent Groovy interpolation
}
```

```bash
# 6. Audit other pipelines for similar patterns
# Search all Jenkinsfiles for hardcoded secrets

# 7. Document the incident
```

---

### 26. How do you prevent unauthorized Groovy script execution in Jenkins?

**Answer:**

```bash
# 1. Lock down the Script Console
# Manage Jenkins → Configure Global Security
# Only Administrators should have "Overall/Run Scripts" permission

# 2. Enable Script Security plugin (enabled by default)
# Untrusted pipelines go through script approval
# Manage Jenkins → In-process Script Approval

# 3. Use sandbox for pipeline scripts
# Declarative pipelines run in sandbox by default
# Sandbox restricts which Groovy methods can be called

# 4. Approve scripts carefully
# Review every script approval request before approving

# 5. Use Shared Libraries with @NonCPS where needed
# Shared Libraries from trusted sources run outside sandbox
```

---

## Deployment & Release Scenarios

### 27. How would you set up a Jenkins pipeline to deploy to multiple environments?

**Answer:**

```groovy
pipeline {
    agent any
    parameters {
        choice(name: 'ENVIRONMENT',
               choices: ['dev', 'staging', 'production'],
               description: 'Target environment')
    }

    environment {
        // Environment-specific config
        APP_URL = "${params.ENVIRONMENT == 'production' ? 'app.example.com' : "${params.ENVIRONMENT}.example.com"}"
    }

    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Deploy') {
            steps {
                withCredentials([
                    sshUserPrivateKey(credentialsId: "${params.ENVIRONMENT}-ssh-key",
                                     keyFileVariable: 'SSH_KEY')
                ]) {
                    sh """
                        ssh -i $SSH_KEY deploy@${APP_URL} \
                            'deploy.sh --version ${BUILD_NUMBER}'
                    """
                }
            }
        }

        stage('Production Approval') {
            when { expression { params.ENVIRONMENT == 'production' } }
            input {
                message "Deploy to production?"
                submitter "release-managers,admin"
            }
            steps {
                echo "Production deployment approved"
            }
        }

        stage('Smoke Test') {
            steps {
                sh "smoke-test.sh --url https://${APP_URL}"
            }
        }
    }
}
```

---

### 28. How would you implement a rolling deployment using Jenkins?

**Answer:**

```groovy
pipeline {
    agent any
    stages {
        stage('Build & Test') {
            steps {
                sh 'mvn clean package'
                sh 'mvn test'
            }
        }

        stage('Deploy to Staging') {
            steps {
                sh 'kubectl set image deployment/myapp myapp=myapp:${BUILD_NUMBER} -n staging'
                sh 'kubectl rollout status deployment/myapp -n staging --timeout=5m'
            }
        }

        stage('Staging Smoke Tests') {
            steps {
                sh 'smoke-test.sh --env staging'
            }
        }

        stage('Production Rolling Deploy') {
            input { message "Proceed to production?" }
            steps {
                // Rolling update with max surge and max unavailable
                sh '''
                    kubectl set image deployment/myapp myapp=myapp:${BUILD_NUMBER} -n production
                    kubectl rollout status deployment/myapp -n production --timeout=10m
                '''
            }
        }

        stage('Post-Deploy Verification') {
            steps {
                sh 'health-check.sh --env production'
            }
            post {
                failure {
                    sh 'kubectl rollout undo deployment/myapp -n production'
                    slackSend message: "Production deploy ROLLED BACK!"
                }
            }
        }
    }
}
```

---

### 29. How would you integrate Jenkins with Docker for continuous delivery?

**Answer:**

```groovy
pipeline {
    agent any

    environment {
        REGISTRY = 'registry.example.com'
        IMAGE_NAME = "${REGISTRY}/myapp"
        IMAGE_TAG = "${BUILD_NUMBER}-${GIT_COMMIT.take(7)}"
    }

    stages {
        stage('Build Image') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
            }
        }

        stage('Test Image') {
            steps {
                sh """
                    docker run --rm ${IMAGE_NAME}:${IMAGE_TAG} npm test
                """
            }
        }

        stage('Security Scan') {
            steps {
                sh "trivy image --severity HIGH,CRITICAL ${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                        echo \$DOCKER_PASS | docker login ${REGISTRY} -u \$DOCKER_USER --password-stdin
                        docker push ${IMAGE_NAME}:${IMAGE_TAG}
                        docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest
                        docker push ${IMAGE_NAME}:latest
                    """
                }
            }
        }

        stage('Deploy') {
            steps {
                sh "kubectl set image deployment/myapp myapp=${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }
    }

    post {
        always {
            sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
            cleanWs()
        }
    }
}
```

---

### 30. How would you automate deployment of a Kubernetes cluster using Jenkins?

**Answer:**

```groovy
pipeline {
    agent {
        kubernetes {
            yaml '''
            apiVersion: v1
            kind: Pod
            spec:
              containers:
                - name: kubectl
                  image: bitnami/kubectl:latest
                  command: ['cat']
                  tty: true
                - name: helm
                  image: alpine/helm:latest
                  command: ['cat']
                  tty: true
            '''
        }
    }

    stages {
        stage('Deploy with Helm') {
            steps {
                container('helm') {
                    withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                        sh """
                            helm upgrade --install myapp ./charts/myapp \
                                --namespace production \
                                --set image.tag=${BUILD_NUMBER} \
                                --set replicas=3 \
                                --wait --timeout 5m
                        """
                    }
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                container('kubectl') {
                    withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                        sh '''
                            kubectl get pods -n production -l app=myapp
                            kubectl rollout status deployment/myapp -n production
                        '''
                    }
                }
            }
        }
    }
}
```

---

## Migration & Administration Scenarios

### 31. You need to migrate Jenkins to Kubernetes. How do you approach it?

**Answer:**

```bash
# Phase 1: Audit current state
# - All jobs and their configurations
# - Plugin list and versions
# - Credentials (encrypted with instance secret key — cannot just copy)
# - Shared libraries
# - Agent configurations

# Phase 2: Export configuration
# - Enable JCasC if not already
# - Export system config as YAML
# - Export plugin list to plugins.txt
# - Document job configurations

# Phase 3: Set up new instance in K8s
helm repo add jenkins https://charts.jenkins.io
helm install jenkins jenkins/jenkins \
    --set controller.JCasC.configScripts.jenkins=@jenkins.yaml \
    --set persistence.size=50Gi

# Phase 4: Migrate
# - Import JCasC configuration
# - Install plugins from plugins.txt
# - Recreate credentials (manual — encrypted differently)
# - Set up Kubernetes agent templates

# Phase 5: Parallel running
# Run both old and new instances
# Validate pipelines produce equivalent results
# Fix any differences

# Phase 6: Cutover
# DNS switch to new instance
# Decommission old instance
```

---

### 32. How do you design a CI/CD pipeline for a microservices application?

**Answer:**

```
┌─────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────┐
│   Checkout   │───▶│  Build & Test │───▶│  Docker Build │───▶│   Push   │
│   (Git)      │    │  (per-service)│    │  & Scan       │    │  Registry│
└─────────────┘    └──────────────┘    └──────────────┘    └────┬─────┘
                                                                │
  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐     │
  │  Production  │◀───│   Staging    │◀───│   Deploy to  │◀────┘
  │  (approval)  │    │ (smoke test) │    │   Staging    │
  └──────────────┘    └──────────────┘    └──────────────┘
```

**Key principles:**
- Each microservice gets its **own pipeline** (own Jenkinsfile)
- Use **Shared Libraries** for common steps (build, push, deploy)
- **Parallel stages** for independent tests
- **Artifact versioning** with Git commit hash + build number
- **Approval gates** for production
- **Automated rollback** on smoke test failure

```groovy
// Individual service Jenkinsfile — uses shared library
@Library('company-pipeline-lib') _

microservicePipeline(
    service: 'user-service',
    registry: 'registry.example.com',
    k8sNamespace: 'production',
    slackChannel: '#user-service-deploys'
)
```

---

### 33. How do you set up Jenkins for a CI/CD pipeline deploying to Kubernetes?

**Answer:**

```groovy
pipeline {
    agent any

    environment {
        REGISTRY = 'registry.example.com'
        IMAGE = "${REGISTRY}/myapp:${GIT_COMMIT.take(7)}"
    }

    stages {
        stage('Build & Test') {
            agent {
                docker { image 'maven:3.8-openjdk-17' }
            }
            steps {
                sh 'mvn clean package'
                sh 'mvn test'
            }
        }

        stage('Docker Build & Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'registry-creds',
                    usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh """
                        docker build -t ${IMAGE} .
                        echo \$PASS | docker login ${REGISTRY} -u \$USER --password-stdin
                        docker push ${IMAGE}
                    """
                }
            }
        }

        stage('Deploy to K8s') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh """
                        kubectl set image deployment/myapp myapp=${IMAGE} -n production
                        kubectl rollout status deployment/myapp -n production --timeout=5m
                    """
                }
            }
        }
    }

    post {
        failure {
            withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                sh 'kubectl rollout undo deployment/myapp -n production'
            }
            slackSend message: "Deploy FAILED and rolled back: ${BUILD_URL}"
        }
    }
}
```

---

### 34. How do you track Jenkins pipeline health and failures proactively?

**Answer:**

**1. Prometheus + Grafana (metrics)**
```bash
# Install "Prometheus Metrics" plugin
# Endpoint: <jenkins-url>/prometheus/

# Key metrics to monitor:
# - jenkins_job_building_duration_seconds
# - jenkins_job_count_value
# - jenkins_executor_count_value
# - jenkins_queue_size_value
# - jenkins_node_online_value
```

**2. Alert rules (Grafana)**
```yaml
# Alert when build queue > 10 for 5 minutes
- alert: JenkinsBuildQueueHigh
  expr: jenkins_queue_size_value > 10
  for: 5m

# Alert when failure rate > 30% in last hour
- alert: JenkinsHighFailureRate
  expr: rate(jenkins_runs_failure_total[1h]) / rate(jenkins_runs_total_total[1h]) > 0.3

# Alert when agent offline
- alert: JenkinsAgentOffline
  expr: jenkins_node_online_value == 0
  for: 2m
```

**3. Centralized logging**
```bash
# Forward Jenkins logs to ELK / Splunk
# $JENKINS_HOME/logs/ → Filebeat → Elasticsearch → Kibana
```

**4. Slack/Email notifications per pipeline**
```groovy
post {
    failure {
        slackSend channel: '#jenkins-alerts',
                  message: "FAILED: ${JOB_NAME} #${BUILD_NUMBER}\n${BUILD_URL}"
    }
    fixed {
        slackSend channel: '#jenkins-alerts',
                  message: "FIXED: ${JOB_NAME} #${BUILD_NUMBER}"
    }
}
```

---

### 35. How do you handle flaky builds across the environment?

**Answer:**

```groovy
// 1. Add retry logic for known flaky steps
stage('Integration Test') {
    steps {
        retry(3) {
            sh 'run-integration-tests.sh'
        }
    }
}

// 2. Use unstable result instead of failure for flaky tests
stage('Flaky Tests') {
    steps {
        script {
            def result = sh(script: 'run-flaky-tests.sh', returnStatus: true)
            if (result != 0) {
                unstable("Flaky tests failed")
            }
        }
    }
}
```

**Systematic approach:**
1. **Identify flaky tests** — Track which tests fail intermittently (JUnit plugin history)
2. **Quarantine** — Move flaky tests to a separate stage, don't block deployments
3. **Fix root causes:**
   - Timing dependencies → Add proper waits/retries
   - Shared state → Isolate test environments
   - External dependencies → Mock them
4. **Clean workspace** — Add `cleanWs()` to prevent stale file issues
5. **Consistent agents** — Use Docker agents for identical environments
6. **Monitor flake rate** — Dashboard showing test stability over time

---

### 36. Explain a complex Jenkins pipeline you've designed, highlighting challenges and resolutions.

**Answer:** *(Example answer structure for interview)*

**Pipeline:** Multi-service deployment pipeline for a microservices application with 12 services, deploying to AWS EKS.

**Architecture:**
```groovy
// Shared Library: vars/microserviceDeploy.groovy
// Called by each service's Jenkinsfile
@Library('platform-pipeline@v2.1') _
microserviceDeploy(
    service: 'payment-service',
    language: 'java',
    deployTarget: 'eks',
    qualityGates: true
)
```

**Challenges and Solutions:**

| Challenge | Solution |
|-----------|----------|
| **12 services with similar pipelines** | Shared Library with configurable template |
| **Long build times (45 min)** | Parallel stages, Docker layer caching, Maven cache |
| **Flaky integration tests** | Retry + quarantine strategy, mock external services |
| **Credential sprawl** | Vault integration, folder-scoped credentials |
| **Agent capacity during peak** | Kubernetes dynamic agents via pod templates |
| **Production safety** | Manual approval gate + automated canary with metric validation |
| **Cross-service dependencies** | Contract testing in CI, version pinning |
| **Rollback automation** | Post-deploy smoke tests + automatic `kubectl rollout undo` on failure |

**Result:** Reduced deployment time from 2 hours (manual) to 15 minutes (automated), with 99.5% deployment success rate.
