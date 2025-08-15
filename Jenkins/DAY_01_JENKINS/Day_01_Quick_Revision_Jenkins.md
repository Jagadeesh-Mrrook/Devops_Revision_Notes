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

