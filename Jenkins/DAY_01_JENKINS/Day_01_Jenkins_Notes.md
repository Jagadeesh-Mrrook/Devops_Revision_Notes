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

