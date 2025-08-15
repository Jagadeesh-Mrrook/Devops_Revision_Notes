# Jenkins Quick Revision Notes

## Concept
- Jenkins: open-source automation server for **build, test, deploy**.
- Enables **CI (Continuous Integration)** & **CD (Delivery/Deployment)**.

## Purpose / Use Case
- CI: automatic builds/tests on commits → early bug detection.
- CD: deploy artifacts to Dev/QA/Prod automatically.
- Automates scripts, reports, notifications.

## How it Works
- Code pushed to SCM → triggers Jenkins pipeline → build → test → deploy.
- CI: Pull code → Static analysis → Build → Tests → Artifact repo.
- CD: Artifact → Deploy → Post-deploy tests.

## Common Issues
- Jenkins not starting (port/memory issues)
- Triggers not firing (webhook misconfig)
- Pipeline fails (missing dependencies/scripts)

## Fixes
- Check logs, fix port conflicts, install dependencies.
- Verify SCM webhooks and credentials.

## Tips
- Prefer pipelines over freestyle jobs.
- Use webhooks, separate CI/CD pipelines.
- Monitor build history & logs.
==============================================================================
# Jenkins Quick Revision — Master/Agent Architecture

## Concept
- Master/Agent architecture (Controller/Worker).
- Master: manages jobs, schedules builds, provides UI.
- Agent: executes jobs/builds, handles load.

## Purpose / Use Case
- Parallel builds → faster pipelines.
- Resource management → avoid master overload.
- Different OS/tools for different jobs.

## How it Works
- Master triggers job → assigns to agent → agent runs build → reports back.
- Connect via SSH, JNLP, or Kubernetes pods.

## Common Issues
- Agent fails to connect (network/auth issues).
- Build fails due to missing tools on agent.
- Master overloaded if no agents used.

## Fixes
- Check connectivity, install required tools.
- Add more agents or scale master resources.

## Tips
- Use labels for specific jobs.
- Keep master lightweight.
- Monitor agent health.
- Use dynamic/cloud agents for scaling.
---

## Concept: Jenkins Installation on Linux (Quick Revision)

### Concept
- Jenkins installed on **Linux** servers (Ubuntu, CentOS).  
- Linux preferred for production master nodes.

### Purpose / Use Case
- Master runs CI/CD pipelines.  
- Agents have required tools, connect via SSH/JNLP; Jenkins not installed on agents.

### How it Works / Steps / Syntax
    # Install Java
    sudo apt update
    sudo apt install openjdk-11-jdk -y

    # Add Jenkins repo & key
    wget -q -O - https://pkg.jenkins.io/debian/jenkins.io.key | sudo apt-key add -
    sudo sh -c 'echo deb https://pkg.jenkins.io/debian binary/ > /etc/apt/sources.list.d/jenkins.list'

    # Install Jenkins
    sudo apt update
    sudo apt install jenkins -y

    # Start and enable service
    sudo systemctl start jenkins
    sudo systemctl enable jenkins

    # Check status
    sudo systemctl status jenkins

### Common Issues / Errors
- Port 8080 in use  
- Java not installed/incompatible  
- Service fails due to dependencies or permissions

### Troubleshooting / Fixes
- Change port in `/etc/default/jenkins`  
- Install compatible Java version  
- Check logs: `/var/log/jenkins/jenkins.log`  
- Fix directory permissions

### Best Practices / Tips
- Use **LTS version**  
- Backup `JENKINS_HOME`  
- Monitor logs regularly  
- Keep master lightweight; offload builds to agents

---
# Jenkins New UI - Quick Revision

## Key Features
- Sidebar navigation (replaces top menu)
- Global search bar (fast job/plugin search)
- Build queue & executor status in collapsible panel
- Dark mode support
- Responsive design for mobile/tablet
- Modern job & pipeline pages with better status icons

## Advantages
- Faster navigation
- Cleaner interface
- Better accessibility

## Switch UI
- Manage Jenkins → Appearance → UI Theme

## Issues
- Some plugins not compatible
- Initial navigation confusion
- Dark mode visibility issues

## Best Practices
- Keep Jenkins updated
- Check plugin compatibility
- Report UI bugs to community
---
1. **Concept:**
   - Freestyle: Basic Jenkins job with GUI config.
   - Pipeline: Script-based CI/CD using `Jenkinsfile`.

2. **Purpose / Use Case in Real-World:**
   - Freestyle: Simple, small tasks or jobs without complex logic.
   - Pipeline: Complex, multi-stage automated workflows.

3. **How it Works / Steps / Syntax:**
   - Freestyle: Configure in Jenkins UI → Save → Build.
   - Pipeline: Create `Jenkinsfile` → Add stages → Commit to repo → Jenkins triggers build.

4. **Common Issues / Errors:**
   - Freestyle: Limited scalability, hard to maintain.
   - Pipeline: Syntax errors in `Jenkinsfile`, missing agents.

5. **Troubleshooting / Fixes:**
   - Freestyle: Split large jobs, add post-build steps.
   - Pipeline: Validate `Jenkinsfile` via `pipeline syntax` tool.

6. **Best Practices / Tips:**
   - Use Pipeline for prod CI/CD.
   - Keep Freestyle for quick tests or one-off jobs.
---
# Jenkins Quick Revision — Build Triggers

## Concept
Manual, SCM-based, and scheduled (CRON) build triggers.
Manual → click Build Now.
SCM → triggers on code changes (polling/webhook).
Schedule → triggers based on CRON.

## Purpose / Use Case
Manual → ad-hoc builds or tests.
SCM → continuous integration on code commit.
Schedule → periodic/nightly builds or reports.

## How it Works
Manual → click Build Now → job executes.
SCM → Poll SCM (Jenkins checks repo) or Webhook (SCM notifies Jenkins).
Schedule → CRON syntax schedules builds automatically.

## Common Issues
SCM polling → extra load if too frequent.
Webhook misconfiguration → builds not triggered.
CRON errors → wrong timing or syntax issues.

## Fixes
Adjust poll interval, verify webhook URL and permissions.
Test CRON syntax, use H for load balancing.

## Tips
Prefer webhooks over polling for efficiency.
Combine triggers if needed.
Use manual builds sparingly for testing/debugging.
---
# Jenkins Quick Revision — Home Directory & File Structure

## Concept
JENKINS_HOME stores all Jenkins data: jobs, plugins, workspaces, logs, secrets, config.xml.
Default Linux path: `/var/lib/jenkins`.

## Purpose / Use Case
Centralized storage → backup/restore, job configs, artifacts, credentials.

## How it Works
- `jobs/` → job configs & build history
- `plugins/` → installed plugins
- `workspace/` → job workspace
- `secrets/` → credentials
- `logs/` → system/build logs
- `config.xml` → global Jenkins config

## Common Issues
Permissions, disk full, corrupted configs.

## Fixes
Set correct permissions, monitor disk, restore from backup.

## Tips
Backup regularly, use version control for configs, separate workspace from system disk.

