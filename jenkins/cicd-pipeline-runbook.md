# Jenkins CI/CD Pipeline Runbook

> **What this is:** A step-by-step guide for understanding, running, and troubleshooting Jenkins CI/CD pipelines in an AWS environment. Written for people who are learning Jenkins and want to understand *why* each component exists, not just what to click.

---

## What Problem Does This Solve?

Every time a developer pushes code, someone has to: run tests, build the app, package it, and deploy it to a server. If that's done manually, it's slow, error-prone, and depends on whoever is on duty remembering the right sequence of steps.

Jenkins automates this entire process. When code is pushed to GitHub:

1. Jenkins detects the change (via webhook)
2. Runs automated tests — if they fail, the deploy stops
3. Builds and packages the application
4. Deploys it to AWS (EC2, S3, ECS, etc.)
5. Notifies the team of success or failure

This is called a **CI/CD pipeline** — Continuous Integration / Continuous Delivery. It's the backbone of modern software deployment.

---

## Core Concepts (Read Before Continuing)

| Term | What It Means |
|------|---------------|
| **Pipeline** | The full automated workflow from code commit to deployment |
| **Stage** | A labeled phase in the pipeline (Build, Test, Deploy) |
| **Step** | A single action inside a stage (run a command, call a script) |
| **Jenkinsfile** | The file in your repo that defines the pipeline — lives alongside your code |
| **Agent** | The machine (or container) that runs the pipeline |
| **Node** | A Jenkins worker machine (the controller delegates work to nodes) |
| **Workspace** | The directory on the agent where your code is checked out |
| **Artifact** | A file produced by the build (e.g., a .jar, .zip, or Docker image) |
| **Webhook** | A GitHub notification that tells Jenkins "new code was pushed" |
| **Blue Ocean** | Jenkins' modern UI for viewing pipelines visually |

---

## Prerequisites

### Jenkins server setup (EC2):
```bash
# Install Java (required for Jenkins)
sudo amazon-linux-extras install java-openjdk11 -y

# Add Jenkins repo
sudo wget -O /etc/yum.repos.d/jenkins.repo \
    https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key

# Install Jenkins
sudo yum install jenkins -y

# Start Jenkins
sudo systemctl enable jenkins
sudo systemctl start jenkins

# Get the initial admin password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

> **Why EC2?** Jenkins needs to run somewhere persistent. An EC2 instance (t3.medium minimum recommended) gives you full control. In production, you'd use Jenkins on EKS or Fargate for better scalability.

### AWS credentials for Jenkins:
- Create an IAM user/role with permissions to deploy to your target service (S3, EC2, ECS)
- Store credentials in **Jenkins Credentials Manager** (not in your Jenkinsfile)
- Never hardcode AWS keys in your pipeline code

---

## The Jenkinsfile: Your Pipeline as Code

The Jenkinsfile lives in the root of your code repository. This is called **Pipeline as Code** — your deployment process is version-controlled alongside your application code. If the pipeline breaks, you can see exactly what changed and roll back.

**Basic Jenkinsfile structure:**
```
pipeline {
    agent { ... }         ← Where to run this
    environment { ... }   ← Variables available to all stages
    stages {
        stage('name') {   ← A labeled phase
            steps { ... } ← What to do in that phase
        }
    }
    post { ... }          ← What to do after everything finishes
}
```

---

## A Real-World Pipeline: Deploy to AWS S3

This example builds a static web application and deploys it to an S3 bucket.

**`Jenkinsfile`**
```groovy
pipeline {
    // WHERE to run this pipeline
    // 'any' means Jenkins picks any available agent
    // In production you'd specify a label: agent { label 'aws-agent' }
    agent any

    // VARIABLES: defined once, used everywhere
    // Using credentials() keeps secrets out of plain text
    environment {
        AWS_REGION        = 'us-east-1'
        S3_BUCKET         = 'my-app-deployment-bucket'
        APP_NAME          = 'my-web-app'
        // Jenkins Credentials Manager stores this securely
        AWS_CREDENTIALS   = credentials('aws-deploy-credentials')
        // BUILD_NUMBER is auto-injected by Jenkins — increments each run
        DEPLOY_VERSION    = "${env.BUILD_NUMBER}-${env.GIT_COMMIT[0..6]}"
    }

    // OPTIONS: global settings for the pipeline
    options {
        // Fail the build if it runs longer than 30 minutes
        // Prevents stuck builds from blocking your agents forever
        timeout(time: 30, unit: 'MINUTES')

        // Keep only the last 10 builds in Jenkins history
        // Prevents disk from filling up on long-running pipelines
        buildDiscarder(logRotator(numToKeepStr: '10'))

        // Don't allow two builds of this pipeline to run at the same time
        // Prevents race conditions during deployment
        disableConcurrentBuilds()
    }

    stages {

        // ── STAGE 1: CHECKOUT ─────────────────────────────────────
        stage('Checkout') {
            steps {
                // Jenkins automatically checks out your repo here
                // 'checkout scm' uses the repo configured in the job
                checkout scm
                echo "Building version: ${DEPLOY_VERSION}"
                echo "Branch: ${env.GIT_BRANCH}"
            }
        }
        // Why a separate Checkout stage? Makes it visible in the pipeline view.
        // You can see if the problem is in source control vs build vs deploy.

        // ── STAGE 2: INSTALL DEPENDENCIES ─────────────────────────
        stage('Install') {
            steps {
                sh 'npm install'
                // sh = shell command, runs on the agent
            }
        }

        // ── STAGE 3: TEST ──────────────────────────────────────────
        stage('Test') {
            steps {
                sh 'npm test'
            }
            post {
                // Publish test results to Jenkins UI regardless of pass/fail
                // This lets you see which tests failed without reading logs
                always {
                    junit 'test-results/**/*.xml'
                }
            }
        }
        // Why test before build? If tests fail, there's no point building
        // or deploying. Fail fast = save time and protect production.

        // ── STAGE 4: BUILD ─────────────────────────────────────────
        stage('Build') {
            steps {
                sh 'npm run build'
                // Creates the /dist folder with production-optimized files

                // Archive the build output as a Jenkins artifact
                // This lets you download the exact build that was deployed
                archiveArtifacts artifacts: 'dist/**/*', fingerprint: true
            }
        }

        // ── STAGE 5: SECURITY SCAN ─────────────────────────────────
        stage('Security Scan') {
            steps {
                // Check for known vulnerabilities in dependencies
                sh 'npm audit --audit-level=high'
                // --audit-level=high: only fail on high/critical severity
                // Low severity issues create noise without real risk
            }
        }
        // Why include a security stage? This is what separates a
        // DevOps pipeline from a DevSecOps pipeline. Demonstrates
        // security awareness — important for cloud/ops roles.

        // ── STAGE 6: DEPLOY ────────────────────────────────────────
        stage('Deploy') {
            // Only deploy from the main branch
            // Feature branches should NOT auto-deploy to production
            when {
                branch 'main'
            }
            steps {
                withAWS(credentials: 'aws-deploy-credentials', region: AWS_REGION) {
                    // Sync build output to S3
                    // --delete removes files in S3 that no longer exist locally
                    sh """
                        aws s3 sync dist/ s3://${S3_BUCKET}/ \
                            --delete \
                            --cache-control max-age=86400
                    """

                    // Invalidate CloudFront cache so users see the new version
                    sh """
                        aws cloudfront create-invalidation \
                            --distribution-id ${CLOUDFRONT_DIST_ID} \
                            --paths '/*'
                    """

                    echo "Deployed version ${DEPLOY_VERSION} to s3://${S3_BUCKET}"
                }
            }
        }
    }

    // POST: runs after all stages complete, regardless of outcome
    post {
        success {
            // Notify team of successful deployment
            // Requires Slack Notification plugin
            slackSend(
                channel: '#deployments',
                color: 'good',
                message: "✅ ${APP_NAME} v${DEPLOY_VERSION} deployed successfully"
            )
        }
        failure {
            slackSend(
                channel: '#deployments',
                color: 'danger',
                message: "❌ ${APP_NAME} build #${BUILD_NUMBER} FAILED — ${BUILD_URL}"
            )
        }
        always {
            // Clean the workspace after every build
            // Prevents old files from affecting future builds
            cleanWs()
        }
    }
}
```

---

## Setting Up the Webhook (GitHub → Jenkins)

The webhook is what makes Jenkins react to code pushes automatically.

### In Jenkins:
1. Go to your pipeline job → **Configure**
2. Under **Build Triggers**, check **GitHub hook trigger for GITScm polling**
3. Note your Jenkins URL: `http://your-jenkins-ip:8080`

### In GitHub:
1. Go to your repo → **Settings** → **Webhooks** → **Add webhook**
2. Payload URL: `http://your-jenkins-ip:8080/github-webhook/`
3. Content type: `application/json`
4. Select: **Just the push event**
5. Click **Add webhook**

> **Why webhooks vs polling?** Jenkins can also poll GitHub every few minutes to check for changes. Webhooks are better — GitHub notifies Jenkins instantly when code is pushed. No lag, no wasted API calls.

---

## Jenkins Credential Management

Never put passwords, API keys, or AWS credentials directly in your Jenkinsfile. Use Jenkins Credentials Manager.

### Add credentials:
1. **Dashboard** → **Manage Jenkins** → **Credentials**
2. Click the appropriate scope (usually "Global")
3. **Add Credentials** → choose type:
   - **Username/Password** — for Docker Hub, GitHub tokens
   - **Secret text** — for API keys, tokens
   - **AWS Credentials** — for AWS access key + secret

### Reference in Jenkinsfile:
```groovy
// Single secret
environment {
    API_KEY = credentials('my-api-key-id')
}

// AWS credentials (creates AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY)
environment {
    AWS_CREDS = credentials('aws-deploy-credentials')
}

// Inside a step (scoped to that block only)
withCredentials([string(credentialsId: 'slack-token', variable: 'SLACK_TOKEN')]) {
    sh "curl -H 'Authorization: Bearer ${SLACK_TOKEN}' ..."
}
```

> **Why Credentials Manager?** Secrets stored here are masked in logs (shown as `****`). They're encrypted at rest. And they're managed centrally — rotate one credential and every pipeline that uses it gets the new value automatically.

---

## Troubleshooting Failed Builds

### Step 1: Find the failure
- **Jenkins Dashboard** → click the failed build (red ball)
- Click **Console Output** to see the full log
- Scroll to the bottom — errors are almost always at the end

### Stage failed, but which command?
- Click the failed stage in the pipeline visualization
- Or search the console output for `ERROR` or `FAILED`

---

### Common Errors & Fixes

**`No agent available`**
```
Waiting for next available executor...
```
- All agents are busy → wait, or add more agents
- Agent is offline → **Manage Jenkins** → **Nodes** → bring the node online
- No agent matches the label you specified → check `agent { label '...' }`

**`Permission denied` on shell commands**
```
chmod: changing permissions of '/var/lib/jenkins': Permission denied
```
- The Jenkins user doesn't have permission to access that path
- Fix: `sudo chown -R jenkins:jenkins /path/to/directory`
- Or: run the command with `sudo` (requires configuring sudoers for jenkins user)

**`Git: Authentication failed`**
```
remote: Invalid username or password
```
- GitHub credentials in Jenkins are wrong or expired
- Fix: Update the credential in Credentials Manager
- GitHub now requires personal access tokens, not passwords

**`aws: command not found`**
```
/tmp/jenkins-script.sh: aws: command not found
```
- AWS CLI not installed on the agent
- Fix: Install AWS CLI on the agent: `pip install awscli`
- Or use a Docker agent with AWS CLI pre-installed

**`Build was aborted`**
```
Timeout has been exceeded
```
- Pipeline ran longer than the timeout set in `options { timeout(...) }`
- Fix: Increase timeout, or find why the step is hanging (usually a hanging test or network call)

**`unstable` build status**
- Tests ran but some failed
- Build and deploy may still proceed (depends on your `when` conditions)
- Check the test results report in Jenkins UI

---

## Pipeline Visualization: Blue Ocean

Blue Ocean is Jenkins' modern UI that shows your pipeline as a visual flowchart.

### Access it:
`http://your-jenkins-ip:8080/blue/`

### What you can see:
- Each stage shown as a box with pass/fail status
- Time taken per stage
- Which specific step failed and the error message
- Branch pipelines side by side

> Blue Ocean makes it easy to explain your pipeline to non-technical stakeholders — worth knowing for interviews.

---

## Integrating Jenkins + Ansible

This is where the two runbooks connect. A common pattern:

1. **Jenkins** builds and packages the application
2. **Jenkins** calls an **Ansible** playbook to configure the target servers
3. **Jenkins** deploys the application to the now-configured servers

```groovy
stage('Configure Servers') {
    steps {
        // Jenkins triggers Ansible to harden/configure EC2 before deploying
        sh """
            ansible-playbook \
                -i inventory/aws_ec2.yml \
                --limit env_production \
                hardening.yml
        """
    }
}

stage('Deploy Application') {
    steps {
        sh 'aws s3 sync dist/ s3://${S3_BUCKET}/'
    }
}
```

> **Why this pattern?** Jenkins handles the "what code goes out" decision. Ansible handles the "what state the servers are in" concern. Separation of responsibilities — each tool does what it's best at.

---

## Key Plugins to Know

| Plugin | What It Does |
|--------|-------------|
| **Pipeline** | Core pipeline support (usually pre-installed) |
| **Git** | Checkout from GitHub/GitLab |
| **AWS Steps** | `withAWS()` block for AWS credentials |
| **Slack Notification** | Send build status to Slack |
| **Blue Ocean** | Visual pipeline UI |
| **Ansible** | Run Ansible playbooks from Jenkins |
| **Docker Pipeline** | Build and run Docker containers in pipelines |
| **JUnit** | Parse and display test results |

Install plugins: **Manage Jenkins** → **Plugins** → **Available plugins**

---

## Security Best Practices

- **Never run Jenkins as root** — use the `jenkins` system user
- **Use matrix-based security** — don't give all users admin access
- **Pin plugin versions** — automatic plugin updates can break pipelines
- **Use agents, not the controller** — don't run builds on the Jenkins controller itself
- **Enable HTTPS** — put Jenkins behind nginx/ALB with TLS
- **Restrict GitHub webhook IPs** — only accept webhooks from GitHub's IP ranges

---

*Part of the [aws-ops-runbooks](../README.md) repository — see also the [Ansible EC2 Hardening Runbook](../ansible/ec2-hardening-runbook.md).*
