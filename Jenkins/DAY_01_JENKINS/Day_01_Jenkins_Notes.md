# Jenkins Notes

## Concept
- Jenkins is an open-source automation server used to **build, test, and deploy applications**.
- Enables **Continuous Integration (CI)** and **Continuous Delivery/Deployment (CD)**.

## Purpose / Use Case in Real-World
- Automates software delivery process for faster, repeatable, and reliable builds.
- **CI:** Detect integration issues early by running automated builds/tests on each commit.
- **CD:** Automatically deploy code to dev, QA, or production environments.
- Automates tasks like generating reports, running scripts, or sending notifications.

## How it Works / Steps / Syntax
- Developer pushes code to **SCM** (GitHub, GitLab, etc.).
- Jenkins pipeline can be triggered **automatically** (commit, PR) or **manually**.
- Steps in CI pipeline:
  1. Pull code
  2. Run static analysis
  3. Build
  4. Run tests
  5. Push artifact to repository (e.g., Artifactory)
- Steps in CD pipeline:
  1. Take artifact
  2. Deploy to environment
  3. Run post-deployment tests

## Common Issues / Errors
- Jenkins server fails to start (port conflict, memory issues)
- Build triggers don’t fire (SCM webhooks misconfigured)
- Pipeline fails due to missing dependencies or script errors

## Troubleshooting / Fixes
- Check Jenkins logs (`/var/log/jenkins/jenkins.log`) for startup errors
- Verify webhook configuration in SCM and Jenkins credentials
- Ensure required build tools/dependencies are installed on agents

## Best Practices / Tips
- Keep Jenkins updated to latest stable version
- Use pipelines (Declarative or Scripted) over freestyle jobs for better maintainability
- Separate CI and CD pipelines for clarity
- Use SCM webhooks for automatic builds, avoid polling
- Monitor build history and logs for early detection of issues
====================================================================================
# Jenkins Notes — Master/Agent Architecture

## Concept
- Jenkins uses a **Master/Agent (Controller/Worker) architecture**.
- **Master (Controller):** Orchestrates tasks, manages jobs, schedules builds, provides the UI, and monitors agents.
- **Agents (Workers):** Execute the actual build tasks, tests, and deployments.

## Purpose / Use Case in Real-World
- **Scalability:** Multiple agents run builds in parallel, speeding up CI/CD pipelines.
- **Resource management:** Heavy builds are offloaded to agents instead of overloading the master.
- **Environment separation:** Different agents can have different OS, tools, or configurations for specific builds.

## How it Works / Steps / Syntax
- Master receives a build request (manual, SCM trigger, or scheduled).
- Master allocates the job to an available agent based on labels or configuration.
- Agent executes the build, runs tests, and reports results back to master.
- Master updates build history, dashboard, and sends notifications.
- Agents can connect via **SSH**, **JNLP**, or **Kubernetes pods** in cloud setups.

## Common Issues / Errors
- Agent fails to connect to master (network issues, firewall, authentication problems).
- Job assigned to agent fails due to missing tools or dependencies.
- Master overloaded if too many jobs run locally without agents.

## Troubleshooting / Fixes
- Check agent logs and connectivity (SSH keys, firewall, master URL).
- Ensure agent environment has required tools installed.
- Add more agents or increase master resources to handle load.

## Best Practices / Tips
- Use labels to assign specific jobs to suitable agents.
- Keep master lightweight; delegate builds to agents.
- Monitor agent health regularly; remove idle or failing agents.
- Use dynamic agents (Kubernetes, AWS EC2) for automatic scaling in cloud setups.
---

## Concept: Jenkins Installation on Linux

### Concept
- Jenkins can be installed on **Linux** servers (Ubuntu, CentOS, etc.).
- Most real-world production Jenkins servers run on Linux for stability and resource efficiency.

### Purpose / Use Case in Real-World
- Linux is the preferred OS for Jenkins master in production setups.
- Jenkins automates CI/CD pipelines by building, testing, and deploying applications.
- Worker nodes (agents) have the required OS and tools installed, but Jenkins itself is not installed on them. Agents connect to master via SSH or JNLP.

### How it Works / Steps / Syntax (Ubuntu Example)
    # Install Java (Jenkins requires Java)
    sudo apt update
    sudo apt install openjdk-11-jdk -y

    # Add Jenkins repository & key
    wget -q -O - https://pkg.jenkins.io/debian/jenkins.io.key | sudo apt-key add -
    sudo sh -c 'echo deb https://pkg.jenkins.io/debian binary/ > /etc/apt/sources.list.d/jenkins.list'

    # Install Jenkins
    sudo apt update
    sudo apt install jenkins -y

    # Start and enable Jenkins service
    sudo systemctl start jenkins
    sudo systemctl enable jenkins

    # Check Jenkins status
    sudo systemctl status jenkins

### Common Issues / Errors
- **Port 8080 already in use:** Jenkins won’t start.  
- **Java not installed or incompatible version:** Jenkins fails to launch.  
- **Service fails to start:** Missing dependencies or directory permission issues.

### Troubleshooting / Fixes
- Change Jenkins port in `/etc/default/jenkins` if 8080 is occupied.  
- Install compatible Java version (Java 11 recommended).  
- Check Jenkins logs: `/var/log/jenkins/jenkins.log`.  
- Ensure proper permissions for Jenkins directories (`/var/lib/jenkins`, `/var/log/jenkins`).

### Best Practices / Tips
- Use **LTS (Long-Term Support) version** for production stability.  
- Backup `JENKINS_HOME` regularly.  
- Monitor logs to detect startup or runtime issues early.  
- Keep Jenkins master lightweight; offload heavy builds to agents.

---
# Jenkins New UI Concepts (Detailed Notes)

## 1. Overview
- Jenkins introduced a **modernized UI** to improve usability, accessibility, and navigation.
- Designed to be more responsive and intuitive for both new and experienced users.
- Available in recent Jenkins versions — can be toggled between Classic UI and New UI.

---

## 2. Key Features of the New UI

### a) Modern Dashboard Layout
- Cleaner design with improved spacing and typography.
- Sidebar navigation replaces the top horizontal menu.
- Dark mode support for better visual comfort.

### b) Global Search
- Located at the top of the sidebar.
- Allows quick navigation to jobs, builds, plugins, and settings.
- Supports partial matches and predictive search.

### c) Build Queue & Executor Status
- Positioned in a collapsible panel.
- Displays queued jobs and currently running jobs.
- Executor details visible directly without page reload.

### d) Responsive Design
- Adapts to mobile and tablet screens.
- Menu collapses into a hamburger-style icon for smaller devices.

### e) Job & Pipeline Pages
- Job status indicators (success, failure, unstable) with modern icons.
- Improved logs viewer with real-time updates.
- Pipeline stage visualization remains intact with better colors and layout.

### f) User Profile & Account Menu
- Accessible from the top-right corner.
- Includes options: `Configure`, `My Views`, `Credentials`, and `Logout`.

---

## 3. Advantages
- Better accessibility (supports screen readers).
- Faster navigation through the sidebar.
- Cleaner, less cluttered interface.

---

## 4. Switching Between Classic & New UI
- Go to **Manage Jenkins → Appearance → UI Theme**.
- Choose between `Classic` and `Modern`.
- Some plugins might still show the classic UI pages.

---

## 5. Common Issues with New UI
- Certain older plugins may not render correctly.
- Users might find initial navigation confusing.
- Dark mode may cause text visibility issues for some custom themes.

---

## 6. Best Practices
- Always use the latest Jenkins version to ensure maximum compatibility.
- If a plugin doesn’t support the new UI, check for an update or raise an issue.
- Provide feedback to Jenkins community for UI improvements.

---
1. Concept:  
   - **Freestyle Job**: A basic Jenkins job type that allows you to build, test, and deploy projects with simple configuration via GUI.  
   - **Pipeline Job**: A Jenkins job that uses a scripted or declarative Jenkinsfile to define the entire CI/CD pipeline as code.  

2. Purpose / Use Case in Real-World:  
   - **Freestyle Job**: Quick setup for simple builds (e.g., running a shell script, triggering builds on commit).  
   - **Pipeline Job**: Complex workflows, multi-stage CI/CD, version-controlled build definitions, easier to replicate across environments.  

3. How it Works / Steps / Syntax:  
   - **Freestyle Job**:  
     1. Go to Jenkins → New Item → Select “Freestyle project”.  
     2. Configure Source Code Management (Git/SVN).  
     3. Add build steps (e.g., Execute shell).  
     4. Save & Build.  
   - **Pipeline Job**:  
     1. Go to Jenkins → New Item → Select “Pipeline”.  
     2. Define pipeline script in GUI or point to `Jenkinsfile` in SCM.  
     3. Use **Declarative Syntax** example:  
        ```groovy
        pipeline {
            agent any
            stages {
                stage('Build') {
                    steps {
                        echo 'Building...'
                    }
                }
                stage('Deploy') {
                    steps {
                        echo 'Deploying...'
                    }
                }
            }
        }
        ```  

4. Common Issues / Errors:  
   - Freestyle jobs: Missing build tools in the Jenkins agent.  
   - Pipeline jobs: Syntax errors in Jenkinsfile, wrong branch/SCM path, missing pipeline plugin.  

5. Troubleshooting / Fixes:  
   - Freestyle: Install required build tools, check job logs for missing dependencies.  
   - Pipeline: Validate Jenkinsfile using "Replay" or Jenkins Linter, ensure pipeline plugin is up-to-date.  

6. Best Practices / Tips:  
   - Prefer pipeline jobs for maintainability and version control.  
   - Keep Jenkinsfile in repo root for easy discovery.  
   - For Freestyle, use parameterized builds for flexibility.  
   - Always test pipeline scripts in a sandbox branch before production.  

---

# Running Builds in Jenkins

## 1. Manual Build Trigger
- Initiated by clicking **"Build Now"** in Jenkins.
- Used for:
  - Testing pipeline changes.
  - Running ad-hoc builds.
  - Immediate deployment needs.

## 2. On SCM Changes
- Triggered automatically when source code changes are detected.
- Configurable in job settings:
  - **Poll SCM**: Jenkins checks the repository periodically (based on CRON syntax).
  - **Webhooks**: SCM (e.g., GitHub, GitLab) notifies Jenkins instantly when changes occur.
- Useful for:
  - Continuous Integration (build on every code commit).

## 3. Scheduled Builds (CRON)
- Uses Jenkins CRON syntax to define schedule.
- Format: `MINUTE HOUR DOM MONTH DOW`
  - Example: `H/15 * * * *` → Every 15 minutes.
- Useful for:
  - Nightly builds.
  - Regular health checks.
  - Periodic deployments.

---
**Key Tip:** These triggers can be combined (e.g., run nightly AND on every commit).
---
# Jenkins Home Directory & File Structure

## Concept
JENKINS_HOME is the main directory where Jenkins stores all configuration, jobs, plugins, workspaces, logs, secrets, and build artifacts.
Default Linux location: `/var/lib/jenkins`.

## Purpose / Use Case
- Centralized storage for all Jenkins data.
- Backup and restore Jenkins server easily.
- Maintain job configurations, build history, plugins, and credentials.

## How it Works / Steps / Structure
Key subdirectories:
- `jobs/` → Job definitions and build history.
- `plugins/` → Installed plugins (`.hpi` or `.jpi` files).
- `workspace/` → Active job workspace for code checkout and builds.
- `secrets/` → Stored credentials and secret keys.
- `logs/` → System and build logs.
- `config.xml` → Global Jenkins configuration.
- Each job has its own `config.xml` inside `jobs/<job_name>/`.

Jenkins reads and updates this directory at startup and during job execution.

## Common Issues / Errors
- Incorrect permissions → Jenkins cannot read/write files.
- Disk full → Builds fail or cannot store artifacts.
- Corrupted `config.xml` → Jenkins startup or job failures.

## Troubleshooting / Fixes
- Set correct OS-level permissions for Jenkins user.
- Monitor disk usage and clean old builds/artifacts.
- Backup `JENKINS_HOME` regularly.
- Restore corrupted configs from backups if needed.

## Best Practices / Tips
- Use a dedicated directory for `JENKINS_HOME`.
- Separate workspace/artifact storage from system disk if possible.
- Enable automated backups.
- Use version control for global `config.xml` and job configurations.
- Be careful when sharing job `config.xml` externally (credentials may be included).

---
