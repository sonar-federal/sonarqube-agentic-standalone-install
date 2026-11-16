# SonarQube Server Enterprise Edition with the agentic pack on one Linux host

This repository holds the installation guide for running SonarQube Server 2026.5 Enterprise Edition together with the SonarQube AI agents on a single Linux host with Docker Compose.

The full procedure, with every command, is in [standalone-install.md](standalone-install.md).

## What is deployed

A generator script from the [docker-sonarqube](https://github.com/SonarSource/docker-sonarqube/tree/master/agentic-compose-generator) repository writes a Docker Compose bundle. The bundle runs these services on one host:

| Component | Role |
|---|---|
| SonarQube Server Enterprise Edition | The analysis server and web interface |
| PostgreSQL | The bundled database, intended for evaluation. Production deployments use `--db external` |
| Agent Orchestrator | Dispatches jobs to the agent runtimes, validates results and creates issues or pull requests |
| Hunter Agent runtime | Runs the LLM-driven scan for flaws that rule-based analysis does not detect |
| Remediation Agent runtime | Proposes a fix for selected issues and opens a pull request on the project's repository |
| Vortex analysis | The analysis service SonarQube uses to verify code with agentic tooling |
| Egress proxy | The only route from the agent runtimes to the internet, restricted to the LLM hosts passed with `--llm-domains` |

The agent runtimes execute code influenced by a language model, so they run in a gVisor sandbox. A runtime that cannot start in the sandbox does not fall back to running without it.

## Before starting

The guide's requirements table lists the host, software, network, license and account prerequisites. In short:

- A Linux host with 8 vCPU, 32 GiB of memory and 150 GB of disk, x86-64 or ARM64, with root access
- An Enterprise Edition license that includes the Hunter Agent and Remediation Agent subscriptions
- A GitHub App for the repositories the agents work on, and an API key for the LLM provider
- Outbound access to Docker Hub, the gVisor download site, the LLM provider and GitHub

## Steps

1. Install Docker Engine and Docker Compose.
2. Install and register the gVisor sandbox (`runsc`) as a Docker runtime.
3. Set `vm.max_map_count` to at least 262144.
4. Clone the generator.
5. Create the settings encryption key and generate the bundle.
6. Start the bundle with `docker compose up -d`.
7. Open SonarQube on port 9443 and sign in.
8. Apply the license, register the LLM provider, connect GitHub and turn on the agents.
9. Check that the agents are connected under **Administration** > **System**.

The guide also covers upgrading and a troubleshooting table for first-start errors.

## Related documentation

- [Before you start](https://docs.sonarsource.com/sonarqube-server/server-installation/ai-agents/before-you-start)
- [Setting up the sandbox runtime](https://docs.sonarsource.com/sonarqube-server/server-installation/ai-agents/sandbox-runtime)
- [Deploying with a Docker installation](https://docs.sonarsource.com/sonarqube-server/server-installation/ai-agents/deploying-with-docker-compose)
- [Deploying alongside a ZIP installation](https://docs.sonarsource.com/sonarqube-server/server-installation/ai-agents/deploying-alongside-zip-installation)
- [Verifying the deployment](https://docs.sonarsource.com/sonarqube-server/server-installation/ai-agents/verifying-the-deployment)

## Tested on

RHEL 9.8 (x86-64) with SonarQube Server 2026.5.4 Enterprise Edition, Docker Compose v5.7.1 and gVisor release 20260706, on 2026-11-16, through step 7 and the service health checks of step 9.

## License

Apache License 2.0. See [LICENSE](LICENSE).
