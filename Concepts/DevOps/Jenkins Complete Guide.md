---
tags:
  - ci-cd
  - devops
---

### For Developers in a GitLab → Jenkins → Quay → ArgoCD → OpenShift Pipeline

## Before You Start: Prerequisites

You don't need to be a Jenkins admin. But you should be comfortable with:

|Prerequisite|Why You Need It|
|---|---|
|Basic Git & GitLab|Jenkins triggers build from your repo|
|Basic Linux CLI|Reading logs, understanding paths|
|What Docker/containers are|Jenkins builds and pushes images to Quay|
|What CI/CD means conceptually|So the pipeline makes sense end-to-end|

**You do NOT need to know:** Groovy deeply, Jenkins internals, or how to set up Jenkins itself.

---

## How Jenkins Fits in Your Pipeline

```
Developer pushes code to GitLab
        ↓
GitLab triggers Jenkins (via webhook)
        ↓
Jenkins runs the Jenkinsfile from your repo
  [Checkout] → [Build] → [Test] → [Push image to Quay]
        ↓
ArgoCD detects new image in Quay
        ↓
ArgoCD deploys to OpenShift
```

**Key mental model:** Jenkins reads a file called `Jenkinsfile` that lives **inside your repo**. This file is the instruction manual for your build. You own it. You can edit it.

---

## Part 1: Jenkins Pipeline Basics

### What is a Pipeline?

A pipeline is an automated sequence of steps that takes your code from commit to deployed artifact. In Jenkins, you define this pipeline as code in a `Jenkinsfile`.

### The Two Pipeline Styles

You will encounter **Declarative** syntax (modern, preferred) and occasionally **Scripted** (older). Focus on Declarative.

```groovy
// DECLARATIVE (what you'll use 95% of the time)
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                echo 'Building...'
            }
        }
    }
}
```

```groovy
// SCRIPTED (older style, you may see this in legacy Jenkinsfiles)
node {
    stage('Build') {
        echo 'Building...'
    }
}
```

---

### The Core Structure

```
pipeline
├── agent          → WHERE to run (which machine/container)
├── environment    → ENV variables available to all stages
├── options        → Pipeline-level settings (timeout, retry, etc.)
├── stages         → The main work — a container for stage blocks
│   ├── stage('Checkout')
│   ├── stage('Build')
│   ├── stage('Test')
│   └── stage('Push to Quay')
└── post           → What to do AFTER pipeline finishes
    ├── always     → Runs no matter what
    ├── success    → Runs only on success
    └── failure    → Runs only on failure
```

---

### Stages

A **stage** is a named phase in your pipeline. It groups related work together and shows up as a labeled box in the Jenkins UI.

```groovy
stages {
    stage('Checkout') {
        steps {
            // Clone the repo
            checkout scm
        }
    }

    stage('Build') {
        steps {
            // Run Maven build
            sh 'mvn clean package -DskipTests'
        }
    }

    stage('Unit Tests') {
        steps {
            sh 'mvn test'
        }
        post {
            always {
                // Always publish test results even if tests fail
                junit 'target/surefire-reports/*.xml'
            }
        }
    }

    stage('Build Docker Image') {
        steps {
            sh 'docker build -t my-app:${BUILD_NUMBER} .'
        }
    }

    stage('Push to Quay') {
        steps {
            withCredentials([usernamePassword(
                credentialsId: 'quay-credentials',
                usernameVariable: 'QUAY_USER',
                passwordVariable: 'QUAY_PASS'
            )]) {
                sh 'docker login quay.io -u $QUAY_USER -p $QUAY_PASS'
                sh 'docker push quay.io/myorg/my-app:${BUILD_NUMBER}'
            }
        }
    }
}
```

**Rules about stages:**

- Stages run **sequentially** by default (top to bottom)
- If one stage fails, subsequent stages are **skipped** (by default)
- Each stage's result (pass/fail/skip) is shown visually in Jenkins UI

---

### Steps

**Steps** are the actual commands inside a stage. Common ones:

|Step|What it does|
|---|---|
|`sh 'command'`|Run a shell command (Linux)|
|`echo 'message'`|Print to console log|
|`checkout scm`|Clone the Git repo that triggered the build|
|`withCredentials([...])`|Inject secrets safely|
|`dir('path')`|Change directory|
|`script { }`|Write Groovy code inside declarative pipeline|
|`stash` / `unstash`|Pass files between stages|

```groovy
steps {
    // Run a shell command
    sh 'mvn clean install'

    // Multi-line shell script
    sh '''
        echo "Building image"
        docker build -t myapp .
        docker tag myapp quay.io/myorg/myapp:latest
    '''

    // Print something
    echo "Build number is: ${env.BUILD_NUMBER}"

    // Change to a subdirectory
    dir('backend') {
        sh 'npm install && npm run build'
    }
}
```

---

### Agent

The `agent` block tells Jenkins **where** to run the pipeline.

```groovy
// Run on any available Jenkins node
agent any

// Run in a Docker container
agent {
    docker {
        image 'maven:3.8-openjdk-17'
        args '-v $HOME/.m2:/root/.m2'  // mount Maven cache
    }
}

// Run on a specific labeled node
agent {
    label 'linux-builder'
}

// No global agent — each stage defines its own
agent none
```

In your company's setup, the agent is likely a specific node or Docker image defined by your team. You usually don't change this.

---

### Environment Variables

```groovy
pipeline {
    environment {
        // Define your own
        APP_NAME = 'my-service'
        QUAY_REGISTRY = 'quay.io/myorg'
        IMAGE_TAG = "${env.BUILD_NUMBER}"   // use Jenkins built-in
    }

    stages {
        stage('Build') {
            steps {
                sh "docker build -t ${QUAY_REGISTRY}/${APP_NAME}:${IMAGE_TAG} ."
            }
        }
    }
}
```

**Jenkins built-in variables you'll see everywhere:**

|Variable|Value|
|---|---|
|`BUILD_NUMBER`|Auto-incrementing build ID (1, 2, 3...)|
|`BUILD_URL`|Full URL to this build in Jenkins UI|
|`JOB_NAME`|Name of the pipeline job|
|`GIT_BRANCH`|Branch that triggered the build|
|`GIT_COMMIT`|Full commit SHA|
|`WORKSPACE`|Path to the job's workspace on the agent|

---

### Post Block

Runs **after** all stages complete.

```groovy
post {
    always {
        // Clean up workspace, publish reports — always runs
        cleanWs()
    }
    success {
        // Notify Slack, send email on success
        slackSend channel: '#builds', message: "✅ Build passed: ${env.JOB_NAME}"
    }
    failure {
        // Alert the team
        slackSend channel: '#builds', message: "❌ Build FAILED: ${env.BUILD_URL}"
        emailext to: 'team@company.com', subject: "FAILED: ${env.JOB_NAME}"
    }
    unstable {
        // Tests passed but with warnings
        echo 'Build is unstable — check test results'
    }
}
```

---

## Part 2: How to Read a Jenkinsfile

### Step-by-Step: Read Top to Bottom

When you open a `Jenkinsfile`, scan it in this order:

1. **`agent`** — Where does this run? What environment?
2. **`environment`** — What variables are set? What registry, what image names?
3. **`stages`** — How many stages? What are their names? This gives you the full picture.
4. **Each stage** — What shell commands run? Any conditions (`when`)?
5. **`post`** — What happens on success/failure?

---

### Real-World Jenkinsfile Example (annotated)

```groovy
pipeline {
    // ① Runs inside a Maven+JDK container — no need to install Maven on the agent
    agent {
        docker { image 'maven:3.8-openjdk-17' }
    }

    // ② Variables used throughout the file
    environment {
        IMAGE_NAME = "quay.io/myorg/backend-service"
        IMAGE_TAG  = "${env.GIT_COMMIT[0..7]}"  // first 8 chars of commit SHA
    }

    stages {
        // ③ Checkout: usually implicit, but sometimes explicit
        stage('Checkout') {
            steps {
                checkout scm  // clones the triggering repo/branch
            }
        }

        // ④ Build the app
        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        // ⑤ Test — notice the post block INSIDE the stage
        stage('Test') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        // ⑥ Build Docker image
        stage('Docker Build') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
            }
        }

        // ⑦ Only push on main branch — see the "when" block
        stage('Push to Quay') {
            when {
                branch 'main'  // ← this stage is SKIPPED on feature branches
            }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'quay-robot-account',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS'
                )]) {
                    sh "docker login quay.io -u $USER -p $PASS"
                    sh "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
                }
            }
        }
    }

    // ⑧ Cleanup always, notify on failure
    post {
        always {
            cleanWs()
        }
        failure {
            slackSend message: "Build failed: ${env.BUILD_URL}"
        }
    }
}
```

---

### Common Patterns to Recognize

**Parallel stages** — stages that run at the same time:

```groovy
stage('Tests') {
    parallel {
        stage('Unit Tests') {
            steps { sh 'mvn test' }
        }
        stage('Integration Tests') {
            steps { sh 'mvn verify -Pintegration' }
        }
    }
}
```

**Conditional execution with `when`:**

```groovy
stage('Deploy') {
    when {
        anyOf {
            branch 'main'
            branch 'release/*'
        }
    }
    steps { ... }
}
```

**Input/approval gate:**

```groovy
stage('Approve Production Deploy') {
    steps {
        input message: 'Deploy to production?', ok: 'Deploy'
    }
}
```

**Using credentials (secrets never hardcoded):**

```groovy
withCredentials([string(credentialsId: 'my-token', variable: 'TOKEN')]) {
    sh 'curl -H "Authorization: Bearer $TOKEN" https://api.example.com'
}
```

---

## Part 3: How to Debug a Failed Build

### Step 1 — Find the Failing Stage

Go to the Jenkins job URL and look at the **Stage View**. Each stage is a colored box:

- 🟢 Green = passed
- 🔴 Red = failed
- ⚪ Grey = skipped

The first red box is your starting point.

---

### Step 2 — Read the Console Log

Click the failed build number → click **Console Output**.

**How to navigate a long log:**

1. Use `Ctrl+F` and search for `ERROR`, `FAILED`, `error:`, or `Exception`
2. Look for the line **just before** it goes red — that's usually the cause
3. Ignore all the green "SUCCESS" lines above the failure

**Common log patterns:**

```
# Compilation error
[ERROR] COMPILATION ERROR :
[ERROR] /workspace/src/Main.java:[42,15] cannot find symbol

# Test failure
Tests run: 10, Failures: 1, Errors: 0
[ERROR] Tests run: 1, Failures: 1 -- in com.example.MyTest
[ERROR] MyTest.shouldReturnUser -- AssertionError: expected 200 but was 404

# Docker/Quay push failure
denied: access forbidden
# → credentials issue or wrong image name

# Shell command not found
mvn: command not found
# → wrong agent/Docker image, Maven not installed

# File not found
ERROR: target/surefire-reports/*.xml: No such file
# → tests didn't run, or ran in wrong directory
```

---

### Step 3 — Match Error to Root Cause

|Symptom in Log|Likely Cause|Fix|
|---|---|---|
|`command not found`|Tool not in agent/image|Check `agent` block, use correct Docker image|
|`Permission denied`|File or socket permission issue|Fix chmod, check Docker socket mount|
|`cannot find symbol` / compile error|Code bug or missing dependency|Fix in code, check `pom.xml`/`package.json`|
|`Connection refused`|External service unreachable|Network issue, wrong URL, service down|
|`access forbidden` on Quay push|Bad credentials or expired token|Update `credentialsId` in Jenkins|
|`No such file or directory`|Wrong working directory|Add `dir('path')` block, check `WORKSPACE`|
|`Tests run: X, Failures: Y`|Unit test failed|Read the test failure message, fix the test or code|
|`OutOfMemoryError`|JVM/container needs more memory|Add `-Xmx` flag to Maven/Java command|

---

### Step 4 — Reproduce Locally (When Needed)

If the error is in your code (not infrastructure), reproduce it locally:

```bash
# Run the same shell command Jenkins ran
mvn clean package

# If Jenkins uses a Docker agent, run the same image
docker run -it --rm -v $(pwd):/workspace maven:3.8-openjdk-17 bash
cd /workspace
mvn clean package
```

This eliminates "it works on my machine" issues.

---

### Step 5 — Replay a Build (Without Committing)

Jenkins has a **Replay** feature — you can edit the Jenkinsfile and re-run **without pushing a commit**:

1. Go to the failed build
2. Click **Replay** in the left sidebar
3. Edit the Jenkinsfile inline
4. Click **Run**

This is excellent for quickly testing fixes to pipeline syntax.

---

### Debugging Checklist

```
[ ] Which stage failed? (Stage View)
[ ] What is the exact error message? (Console Output → search ERROR)
[ ] Is it a code error or a pipeline/infra error?
[ ] Did this pipeline work before? What changed?
[ ] Is it failing only on this branch? Or on main too?
[ ] Are credentials expired? (common cause of push failures)
[ ] Is the Docker image in the agent block correct/available?
[ ] Try Replay with a fix before committing
```

---

## Quick Reference Card

### Jenkinsfile Skeleton

```groovy
pipeline {
    agent { docker { image 'IMAGE' } }

    environment {
        KEY = 'value'
    }

    stages {
        stage('Name') {
            when { branch 'main' }
            steps {
                sh 'command'
            }
        }
    }

    post {
        always  { cleanWs() }
        failure { echo 'Failed!' }
        success { echo 'Done!' }
    }
}
```

### Vocabulary Cheat Sheet

|Term|Meaning|
|---|---|
|**Pipeline**|The whole CI/CD process defined in code|
|**Jenkinsfile**|The file that defines the pipeline (lives in your repo)|
|**Stage**|A named phase: Build, Test, Push|
|**Step**|A single action inside a stage (`sh`, `echo`, etc.)|
|**Agent**|Where the pipeline runs (node, container)|
|**Credentials**|Secrets stored in Jenkins, injected via `withCredentials`|
|**Workspace**|Temp folder on the agent where your code is checked out|
|**Replay**|Re-run a build with an edited Jenkinsfile (no commit needed)|
|**SCM**|Source Control Management = your GitLab repo|
|`BUILD_NUMBER`|Auto-incrementing ID for each run|

---

## What NOT to Learn (Yet)

Skip these until you actually need them:

- **Shared Libraries** — reusable Groovy code across pipelines (advanced)
- **Jenkins DSL / Job DSL** — programmatic job creation (admin territory)
- **Master/agent architecture** — how to set up Jenkins nodes (ops territory)
- **Blue Ocean** — a deprecated Jenkins UI plugin
- **Full Groovy language** — you only need the small subset used in Jenkinsfiles

---

_Focus on reading Jenkinsfiles, understanding stage failures, and using Replay. That covers 90% of what a developer needs day-to-day._