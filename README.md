# SonarQube Server Enterprise Edition with the agentic pack on one Linux host

This repository holds the installation guide for running SonarQube Server 2026.5 Enterprise Edition together with the SonarQube AI agents on a single Linux host with Docker Compose.

The full procedure is in [standalone-install.md](standalone-install.md). This page describes what the deployment contains, what it needs and the order of the steps.

## What the deployment contains

A generator script from the [docker-sonarqube](https://github.com/SonarSource/docker-sonarqube/tree/master/agentic-compose-generator) repository writes a Docker Compose bundle. The bundle runs these services on one host:

| Component | Role |
|---|---|
| SonarQube Server Enterprise Edition | The analysis server and web interface |
| PostgreSQL | The bundled database, intended for evaluation. Production deployments use `--db external` |
| Agent Orchestrator | Stages repositories, dispatches jobs to the agent runtimes, validates results and creates issues or pull requests. Shares SonarQube's database |
| Hunter Agent runtime | Runs the LLM-driven scan for flaws that rule-based analysis does not detect |
| Remediation Agent runtime | Proposes a fix for selected issues and opens a pull request on the project's repository |
| Vortex analysis | The analysis service SonarQube uses to verify code with agentic tooling |
| Egress proxy | The only route from the agent runtimes to the internet, restricted to the LLM hosts passed with `--llm-domains` |

The agent runtimes execute code influenced by a language model, so they run in a gVisor sandbox. A runtime that cannot start in the sandbox does not fall back to running without it.

To connect the agents to a SonarQube Server that is already installed, generate the bundle with `--edition none`. That path is described in [Deploying with Docker Compose](https://docs.sonarsource.com/sonarqube-server/server-installation/ai-agents/deploying-with-docker-compose).

## Requirements

| Item | Requirement |
|---|---|
| Host | Linux, x86-64 or ARM64, kernel 4.14.77 or later. Docker Desktop on macOS or Windows is not supported |
| Size | 8 vCPU, 32 GiB memory and 150 GB of disk for one Hunter runtime and one Remediation runtime. See [Capacity planning](https://docs.sonarsource.com/sonarqube-server/server-installation/ai-agents/capacity-planning) for concurrent jobs |
| Software | Docker Engine with Docker Compose v2.23 or later, Python 3.9 or later, git, and root access |
| Network | Outbound access to Docker Hub, the gVisor download site, the LLM provider and GitHub (or GitHub Enterprise) |
| License | An Enterprise Edition license with Hunter Agent and Remediation Agent subscriptions. See [Before you start](https://docs.sonarsource.com/sonarqube-server/server-installation/ai-agents/before-you-start) |
| Other | A GitHub App for the repositories the agents work on, and an API key for the LLM provider |

## Steps

Each step is described in full, with commands, in [standalone-install.md](standalone-install.md).

1. **Install Docker Engine and Docker Compose.** Commands are given for RHEL, Rocky Linux and AlmaLinux. Other distributions follow the [Docker Engine installation guide](https://docs.docker.com/engine/install/).
2. **Install and register the gVisor sandbox.** Download `runsc`, verify its checksum and register it as the `runsc` runtime in `/etc/docker/daemon.json`.
3. **Set `vm.max_map_count` to at least 262144.** SonarQube's search component requires it.
4. **Get the generator.** Clone `https://github.com/SonarSource/docker-sonarqube.git` and change to the `agentic-compose-generator` directory.
5. **Generate the bundle.** Create the settings encryption key, then run the generator:

   ```
   python3 generate.py --edition enterprise \
     --sonar-secret-key-file ./sonar-secret.txt \
     --llm-domains <LLM provider host name> \
     --out ~/bundle
   ```

6. **Start the bundle.** Run `docker compose up -d` in the bundle directory. The first start takes several minutes.
7. **Open SonarQube** at `https://<host name>:9443`. The default certificate is self-signed.
8. **Configure SonarQube.** Apply the license, confirm the settings encryption key is registered, register the LLM provider and model, connect the repositories through a GitHub App, run an analysis and turn on the agents for the projects.
9. **Check that the agents are connected** under **Administration** > **System**.

## Generator options used

| Option | Meaning |
|---|---|
| `--edition enterprise` | Runs SonarQube Server Enterprise Edition and a bundled PostgreSQL |
| `--sonar-secret-key-file` | The settings encryption key, mounted into SonarQube Server and the orchestrator. The path needs the leading `./` |
| `--llm-domains` | The hosts the agent runtimes may reach through the egress proxy. The default is `api.anthropic.com`. Wildcards are refused |
| `--out` | The directory the bundle is written to |
| `--tls-server-name` | The host name in the bundle's certificate and in SonarQube's server base URL. The default is `localhost` |
| `--sonarqube-tag` | The SonarQube version, for example `2026.5.4` |

The generator README lists every option: [agentic-compose-generator/README.md](https://github.com/SonarSource/docker-sonarqube/blob/master/agentic-compose-generator/README.md).

## Upgrading

Regenerate the bundle into the same directory with the same options and the new `--sonarqube-tag`, then run `docker compose up -d`. The generator keeps the signing secret and stops before overwriting a file that was edited. Local changes belong in `docker-compose.override.yaml`. `docker compose down -v` deletes the named volumes, including the database, and is not used for upgrades.

## Troubleshooting

The troubleshooting table in [standalone-install.md](standalone-install.md#troubleshooting) covers these first-start symptoms:

- `unknown or invalid runtime name: runsc`
- `refers to undefined volume sonar-secret.txt`
- Agent runtimes that cannot resolve `egress-proxy`
- An orchestrator that is unhealthy right after the first start
- Agent jobs that cannot reach the LLM
- `exec format error` on start

## References

- [Before you start](https://docs.sonarsource.com/sonarqube-server/server-installation/ai-agents/before-you-start)
- [Setting up the sandbox runtime](https://docs.sonarsource.com/sonarqube-server/server-installation/ai-agents/sandbox-runtime)
- [Setting up shared storage](https://docs.sonarsource.com/sonarqube-server/server-installation/ai-agents/shared-storage)
- [Deploying with Docker Compose](https://docs.sonarsource.com/sonarqube-server/server-installation/ai-agents/deploying-with-docker-compose)
- [Verifying the deployment](https://docs.sonarsource.com/sonarqube-server/server-installation/ai-agents/verifying-the-deployment)
- [Sensitive settings](https://docs.sonarsource.com/sonarqube-server/instance-administration/security/encrypting-settings)

Content verified against the public documentation and the generator on 2026-11-16.
