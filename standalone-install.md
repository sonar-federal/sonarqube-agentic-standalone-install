# Installing SonarQube Server Enterprise Edition with the agentic pack on one Linux host

This guide installs SonarQube Server 2026.5 Enterprise Edition together with the Agent Orchestrator, the Hunter Agent, the Remediation Agent, Vortex analysis and the egress proxy on a single Linux host with Docker Compose. A generator script produces the Compose bundle, so no Compose file is written by hand.

To connect the agents to a SonarQube Server that is already installed, use `--edition none` instead. That path is covered in [Deploying with Docker Compose](https://docs.sonarsource.com/sonarqube-server/server-installation/ai-agents/deploying-with-docker-compose).

## Requirements

| Item | Requirement |
|---|---|
| Host | Linux, x86-64 or ARM64, kernel 4.14.77 or later. Docker Desktop on macOS or Windows is not supported |
| Size | 8 vCPU, 32 GiB memory and 150 GB of disk hold SonarQube, PostgreSQL, the orchestrator, one Hunter runtime, one Remediation runtime, Vortex and the egress proxy. See [Capacity planning](https://docs.sonarsource.com/sonarqube-server/server-installation/ai-agents/capacity-planning) to size for concurrent jobs |
| Software | Docker Engine with Docker Compose v2.23 or later, Python 3.9 or later, git, and root access on the host |
| Network | Outbound access to Docker Hub, the gVisor download site, your LLM provider and GitHub (or your GitHub Enterprise instance) |
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

For other distributions, follow the [Docker Engine installation guide](https://docs.docker.com/engine/install/).

## 2. Install and register the gVisor sandbox

Download the `runsc` binary, verify its checksum and install it:

```
V=20260706; ARCH=$(uname -m)
URL=https://storage.googleapis.com/gvisor/releases/release/$V/$ARCH
curl -fsSL -o runsc $URL/runsc
curl -fsSL -o runsc.sha512 $URL/runsc.sha512
sha512sum -c runsc.sha512
chmod a+rx runsc && sudo mv runsc /usr/local/bin/
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

Restart Docker and confirm a container starts under gVisor. The first line of the output is `Starting gVisor...`.

```
sudo systemctl restart docker
docker run --runtime=runsc --rm alpine dmesg
```

If the container fails with an SELinux denial, see the gVisor documentation for the SELinux options that apply to your distribution.

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

The output of the last command is `vm.max_map_count = 262144`. The file name starts with `zz-` because the files in `/etc/sysctl.d/` are applied in alphabetical order and the last file wins, so a distribution file with a later name cannot override it.

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

```
cd ~/bundle
docker compose up -d
docker compose ps
```

The first start takes several minutes. The init services, such as `storage-init` and `truststore-init`, never report healthy: they prepare a volume and exit with code 0. `docker compose ps -a` lists them as exited.

## 7. Open SonarQube

Browse to `https://<host name>:9443`. The certificate is self-signed, so the browser shows a warning until a certificate of your own is installed. The bundle README describes how to serve your own certificate.

To reach a host that does not accept direct connections, forward the port over SSH and browse to `https://localhost:9443`:

```
ssh -N -L 9443:localhost:9443 <user>@<host>
```

The command stays in the foreground while the tunnel is open.

Sign in with the default administrator account and change the password when prompted.

## 8. Configure SonarQube

1. Apply the license under **Administration** > **Configuration** > **License Manager**. The license must include the Hunter Agent and Remediation Agent subscriptions.
2. Confirm that the settings encryption key is in place before registering the LLM provider, so that the provider's API key is stored encrypted. Open **Administration** > **Configuration** > **Encryption**. The page reads "Secret key is registered", because the bundle mounted the key file from step 5. Do not select **Generate New Secret Key**: it replaces the key, and the orchestrator keeps decrypting with the old one. See [Sensitive settings](https://docs.sonarsource.com/sonarqube-server/instance-administration/security/encrypting-settings).
3. Register the LLM provider and select a model under **Administration** > **Configuration** > **AI Capabilities** > **LLM Providers**. The agent toggles stay locked until a provider and a model are saved.
4. Connect the repositories through a GitHub App, bind each project to its repository, and run an analysis so that the agents have issues to work from.
5. Turn on the Hunter Agent and the Remediation Agent for the projects.

## 9. Check that the agents are connected

Open **Administration** > **System**. The Agent Orchestrator, Vortex analysis and each agent runtime should report healthy. See [Verifying the deployment](https://docs.sonarsource.com/sonarqube-server/server-installation/ai-agents/verifying-the-deployment).

## Upgrading

Regenerate the bundle into the same directory with the same options and the new `--sonarqube-tag`, then run `docker compose up -d`. The generator keeps the signing secret and stops before overwriting a file that was edited. Local changes belong in `docker-compose.override.yaml`, which the generator never writes. Do not run `docker compose down -v`: it deletes the named volumes, including the database.

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
