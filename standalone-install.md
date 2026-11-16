# Installing SonarQube Server Enterprise Edition with the agentic pack on one Linux host

This guide installs SonarQube Server 2026.5 Enterprise Edition together with the Agent Orchestrator, the Hunter Agent, the Remediation Agent, Vortex analysis and the egress proxy on a single Linux host with Docker Compose. A generator script produces the Compose bundle, so no Compose file is written by hand.

To connect the agents to a SonarQube Server installed from the ZIP file, the generator is run with `--edition none`; see [Deploying alongside a ZIP installation](https://docs.sonarsource.com/sonarqube-server/server-installation/ai-agents/deploying-alongside-zip-installation). The generator cannot attach the agents to a SonarQube Server that already runs in a container. The Docker path is described in [Deploying with a Docker installation](https://docs.sonarsource.com/sonarqube-server/server-installation/ai-agents/deploying-with-docker-compose).

## Requirements

| Item | Requirement |
|---|---|
| Host | Linux, x86-64 or ARM64, kernel 4.14.77 or later. Docker Desktop on macOS or Windows is not supported |
| Size | 8 vCPU, 32 GiB memory and 150 GB of disk is the size this procedure was run on, with SonarQube, PostgreSQL, the orchestrator, one Hunter runtime, one Remediation runtime, Vortex and the egress proxy. It was not load-tested. See [Capacity planning](https://docs.sonarsource.com/sonarqube-server/server-installation/ai-agents/capacity-planning) to size for concurrent jobs |
| Software | Docker Engine with Docker Compose v2.23 or later, Python 3.9 or later, git, openssl, and root access on the host |
| Network | Outbound access to Docker Hub, the gVisor download site, the LLM provider and GitHub (or a GitHub Enterprise instance) |
| License | An Enterprise Edition license, with Hunter Agent and Remediation Agent subscriptions added to it. See [Before you start](https://docs.sonarsource.com/sonarqube-server/server-installation/ai-agents/before-you-start) |
| Other | A GitHub App for the repositories the agents will work on, and an API key for the LLM provider |

The agent runtimes execute code influenced by a language model, so they run in a gVisor sandbox. A runtime that cannot start in the sandbox does not fall back to running without it.

## 1. Install Docker Engine and Docker Compose

On RHEL, Rocky Linux and AlmaLinux:

```
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf -y install docker-ce docker-ce-cli containerd.io docker-compose-plugin git
sudo systemctl enable --now docker
sudo usermod -aG docker "$USER"
```

Log out and back in so the group change applies, then check the versions:

```
docker compose version
docker run --rm hello-world
```

These commands use Docker's CentOS repository, which installed without conflicts on RHEL 9.8. Docker documents a separate RHEL repository (`https://download.docker.com/linux/rhel/docker-ce.repo`) and the removal of conflicting packages such as `podman` and `runc` in its [RHEL installation guide](https://docs.docker.com/engine/install/rhel/). Membership of the `docker` group gives root-equivalent access to the host, so only administrators belong in it. For other distributions, follow the [Docker Engine installation guide](https://docs.docker.com/engine/install/).

## 2. Install and register the gVisor sandbox

Download the `runsc` binary, verify its checksum and install it:

```
V=20260706; ARCH=$(uname -m)
URL=https://storage.googleapis.com/gvisor/releases/release/$V/$ARCH
curl -fsSL -o runsc $URL/runsc
curl -fsSL -o runsc.sha512 $URL/runsc.sha512
sha512sum -c runsc.sha512
chmod a+rx runsc && sudo mv runsc /usr/local/bin/
rm runsc.sha512
```

Register `runsc` as a Docker runtime in `/etc/docker/daemon.json`. If the file already exists, add the `runtimes` entry to it instead of replacing the file.

```
{
  "runtimes": {
    "runsc": {
      "path": "/usr/local/bin/runsc",
      "runtimeArgs": ["--network=host", "--overlay2=root:self,size=50g"]
    }
  }
}
```

`--network=host` lets containers on a user-defined network resolve each other's names under gVisor. `--overlay2=root:self,size=50g` caps the writable layer of each job container.

Restart Docker and confirm a container starts under gVisor. The output includes `Starting gVisor...`. Restarting Docker stops running containers unless `live-restore` is enabled, so this step belongs before the bundle is started.

```
sudo systemctl restart docker
docker run --runtime=runsc --rm alpine dmesg
```

With SELinux in enforcing mode on RHEL 9.8, the container started without a denial. If a container fails with an SELinux denial on another distribution, see [Setting up the sandbox runtime](https://docs.sonarsource.com/sonarqube-server/server-installation/ai-agents/sandbox-runtime) and the gVisor documentation.

## 3. Set the kernel parameter for Elasticsearch

SonarQube's search component needs a memory-map limit of at least 262144. Check the current value:

```
sysctl vm.max_map_count
```

Write the setting to a drop-in file, apply it and confirm the result:

```
echo 'vm.max_map_count=262144' | sudo tee /etc/sysctl.d/zz-sonarqube.conf
sudo sysctl --system
sysctl vm.max_map_count
```

The output of the last command is `vm.max_map_count = 262144`. The file name starts with `zz-` because the files in `/etc/sysctl.d/` are applied in alphabetical order and the last file wins. If the last command reports a different value, another configuration file, such as `/etc/sysctl.conf`, sets the parameter and needs the same value.

## 4. Get the generator

The generator is a single Python file that uses only the standard library:

```
git clone --depth 1 https://github.com/SonarSource/docker-sonarqube.git
cd docker-sonarqube/agentic-compose-generator
python3 generate.py --help
```

## 5. Generate the bundle

Create the settings encryption key before generating the bundle. SonarQube Server and the orchestrator must use the same key, and the generator mounts the file into both. The key is a Base64-encoded AES-256 key:

```
mkdir -p ~/bundle
(umask 077 && openssl rand -base64 32 > ~/bundle/sonar-secret.txt)
```

Generate the bundle:

```
python3 generate.py --edition enterprise \
  --sonar-secret-key-file ./sonar-secret.txt \
  --llm-domains <LLM provider host name> \
  --out ~/bundle
```

Without `--tls-server-name`, the certificate and SonarQube's server base URL use `localhost`, and access is through an SSH tunnel (step 7). To reach the host by a name, add `--tls-server-name <host name>` to the command.

| Option | Meaning |
|---|---|
| `--edition enterprise` | Runs SonarQube Server Enterprise Edition and a bundled PostgreSQL inside the bundle |
| `--sonar-secret-key-file` | The settings encryption key, mounted into SonarQube Server and the orchestrator. The path is relative to the bundle directory and needs the leading `./`; a bare file name is read as a volume name, and `docker compose up` stops with `refers to undefined volume` |
| `--llm-domains` | The only hosts the agent runtimes may reach through the egress proxy. The default is `api.anthropic.com`. Wildcards are refused |
| `--out` | The directory the bundle is written to |
| `--tls-server-name` | Optional. The host name used in the bundle's certificate and in SonarQube's server base URL. The default is `localhost`, which suits access through an SSH tunnel. Set it when users reach the host by a name |
| `--sonarqube-tag` | Optional. The SonarQube version, for example `2026.5.4`; the generator adds the edition suffix. Without it the generator uses the version in its `default-images.json` |

The defaults use local storage and a self-signed TLS certificate. The bundled database is intended for evaluation. For production, use `--db external`. See [the generator README](https://github.com/SonarSource/docker-sonarqube/blob/master/agentic-compose-generator/README.md) for all options.

To review the Compose file before generating the bundle, run the same command with `--out -`. Do not pass `--image-tag` unless the tag exists for all four agentic images, because their default tags differ.

The bundle contains the secret that signs requests between SonarQube, the orchestrator and Vortex. The generator writes it to `.env` with mode 600. Keep the bundle directory private and out of version control.

## 6. Start the bundle

The bundle publishes port 9443 on every network interface of the host, and the default administrator account is reachable on that port until its password is changed (step 7). Restrict inbound access to 9443 with a firewall or security group before starting the bundle.

```
cd ~/bundle
docker compose up -d
docker compose ps
```

The first start takes several minutes. The init services, `storage-init` and `signing-init`, never report healthy: they prepare a volume and exit with code 0. `docker compose ps -a` lists them as exited.

## 7. Open SonarQube

Browse to `https://<host name>:9443`, where the host name is the `--tls-server-name` value, or to `https://localhost:9443` through the SSH tunnel below when the default was used. The certificate is self-signed, so the browser shows a warning until another certificate is installed. The bundle README describes how to serve another certificate.

To reach a host that does not accept direct connections, forward the port over SSH and browse to `https://localhost:9443`:

```
ssh -N -L 9443:localhost:9443 <user>@<host>
```

The command stays in the foreground while the tunnel is open.

Sign in with `admin` and `admin`. SonarQube asks for a new password.

## 8. Configure SonarQube

1. Apply the license under **Administration** > **Configuration** > **License Manager**. The license must include the Hunter Agent and Remediation Agent subscriptions.
2. Confirm that the settings encryption key is in place before registering the LLM provider, so that the provider's API key is stored encrypted. Open **Administration** > **Configuration** > **Encryption**. The page reads "Secret key is registered", because the bundle mounted the key file from step 5. Do not generate a new secret key on this page: it replaces the key, and the orchestrator keeps decrypting with the old one. See [Sensitive settings](https://docs.sonarsource.com/sonarqube-server/instance-administration/security/encrypting-settings).
3. Register the LLM provider and select a model under **Administration** > **Configuration** > **AI Capabilities** > **LLM Providers**. The agent toggles stay locked until a provider and a model are saved.
4. Create a GitHub App integration under **Administration** > **Configuration** > **DevOps Platform Integrations**, bind each project to its repository, and run an analysis so that the agents have issues to work from.
5. Turn on the Hunter Agent and the Remediation Agent under **Administration** > **Configuration** > **AI Capabilities**.

## 9. Check that the agents are connected

Open **Administration** > **System**. The Agent Orchestrator, Vortex analysis and each agent runtime report healthy. See [Verifying the deployment](https://docs.sonarsource.com/sonarqube-server/server-installation/ai-agents/verifying-the-deployment).

## Upgrading

Regenerate the bundle into the same directory with the same options and the new `--sonarqube-tag`, then run `docker compose up -d`. The generator keeps the signing secret and stops before overwriting a file that was edited. After an update that changes the database schema, SonarQube Server does not migrate the database by itself and stays unhealthy until the migration is started: open `/setup` in SonarQube Server, or send `POST /api/system/migrate_db`. The orchestrator starts after the migration. Patch updates within one release line do not normally change the schema. The generator sets `stop_grace_period` on each service so that a stop waits for running jobs to drain, up to 44,430 seconds for the Hunter Agent runtime. Removing it makes a stop kill running jobs after 10 seconds and loses their results; see [Deploying with a Docker installation](https://docs.sonarsource.com/sonarqube-server/server-installation/ai-agents/deploying-with-docker-compose). Local changes belong in `docker-compose.override.yaml`, which the generator never writes. Do not run `docker compose down -v`: it deletes the named volumes, including the database.

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| Docker reports `unknown or invalid runtime name: runsc` | `runsc` is not registered with Docker. Repeat step 2 and restart Docker |
| `docker compose up` stops with `refers to undefined volume sonar-secret.txt` | `--sonar-secret-key-file` was given without a leading `./`. Generate the bundle again with `./sonar-secret.txt`; the generator keeps the signing secret |
| The agent runtimes cannot resolve `egress-proxy`, with a DNS error from 127.0.0.11 | `runsc` was registered without `--network=host` |
| The orchestrator is unhealthy right after the first start | It waits for SonarQube to create the database schema. Wait a few minutes |
| Agent jobs fail to reach the LLM | The host is not in `--llm-domains`. Check the proxy log with `docker compose logs -f egress-proxy` |
| `exec format error` on start | The image architecture does not match the host architecture |

## References

- [Before you start](https://docs.sonarsource.com/sonarqube-server/server-installation/ai-agents/before-you-start)
- [Setting up the sandbox runtime](https://docs.sonarsource.com/sonarqube-server/server-installation/ai-agents/sandbox-runtime)
- [Setting up shared storage](https://docs.sonarsource.com/sonarqube-server/server-installation/ai-agents/shared-storage)
- [Deploying with Docker Compose](https://docs.sonarsource.com/sonarqube-server/server-installation/ai-agents/deploying-with-docker-compose)
- [Agentic Compose generator README](https://github.com/SonarSource/docker-sonarqube/blob/master/agentic-compose-generator/README.md)
