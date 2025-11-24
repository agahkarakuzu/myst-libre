# Docker-in-Docker (DinD) Support for myst-libre

## Overview

This guide explains how to run myst-libre inside a Docker container while enabling it to spawn Jupyter containers on the host machine. This is useful when:

- Your AI agent or orchestration system runs in containers
- You want to keep myst-libre execution isolated
- You need to integrate myst-libre into a containerized workflow
- You want consistent environments across different host machines

## Architecture

```
┌─────────────────────────────────────────────────────┐
│ Host Machine                                        │
│                                                     │
│  Docker Daemon                                      │
│  │                                                  │
│  ├─ myst-libre Container                           │
│  │  ├─ Python & MyST CLI                           │
│  │  ├─ myst-libre package                          │
│  │  └─ /workspace (mounted from host)              │
│  │      └─ /builds, /DATA, /config                 │
│  │                                                  │
│  └─ Jupyter Container (sibling)                    │
│     ├─ Jupyter Server                              │
│     ├─ Notebook Execution Environment              │
│     └─ /home/jovyan (mounted from host)            │
│         └─ Build sources & data                    │
│                                                     │
│  Key: Containers are SIBLINGS (both managed        │
│        by host Docker daemon)                       │
└─────────────────────────────────────────────────────┘
```

## How It Works

### 1. Path Translation

When myst-libre runs in a container, paths have different meanings:

**Inside myst-libre container:**
```
/workspace/builds/user/repo/latest  ← Container path
```

**On the host machine:**
```
/home/user/workspace/builds/user/repo/latest  ← Host path
```

When the myst-libre container spawns a Jupyter container, the Docker daemon interprets volume mount paths relative to the HOST, not the myst-libre container. The `host_path_prefix` parameter automatically translates paths.

### 2. Docker Socket Access

The myst-libre container accesses the host Docker daemon via the socket:
```
/var/run/docker.sock  ← Host Docker socket
```

This is mounted into the myst-libre container, allowing it to spawn containers on the host.

## Managing Secrets: Two Approaches

**CRITICAL:** Never include `.env` files in Docker images. This exposes credentials and violates security best practices.

myst-libre supports two secure approaches for providing credentials:

### Approach 1: Environment Variables (Recommended for CI/CD)

Pass credentials directly as environment variables. This is ideal for:
- CI/CD pipelines (GitHub Actions, GitLab CI, etc.)
- Container orchestration (Kubernetes, Docker Swarm)
- Cloud deployments (AWS ECS, Azure Container Instances, etc.)
- Any automated system where you don't want files in the image

**Supported credentials:**
- `DOCKER_PRIVATE_REGISTRY_USERNAME` - Username for private Docker registries
- `DOCKER_PRIVATE_REGISTRY_PASSWORD` - Password for private Docker registries
- `CURVENOTE_TOKEN` - API token for Curvenote (optional)

**With docker run:**
```bash
docker run -it --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /home/user/workspace:/workspace \
  -e HOST_WORKSPACE_PATH=/home/user/workspace \
  -e DOCKER_PRIVATE_REGISTRY_USERNAME=your_username \
  -e DOCKER_PRIVATE_REGISTRY_PASSWORD=your_password \
  myst-libre:latest \
  python example_docker_in_docker.py
```

**With docker-compose:**
```yaml
environment:
  - HOST_WORKSPACE_PATH=/home/user/workspace
  - DOCKER_PRIVATE_REGISTRY_USERNAME=${DOCKER_PRIVATE_REGISTRY_USERNAME}
  - DOCKER_PRIVATE_REGISTRY_PASSWORD=${DOCKER_PRIVATE_REGISTRY_PASSWORD}
  - CURVENOTE_TOKEN=${CURVENOTE_TOKEN}
```

**Then run:**
```bash
export DOCKER_PRIVATE_REGISTRY_USERNAME=myuser
export DOCKER_PRIVATE_REGISTRY_PASSWORD=mypass
export CURVENOTE_TOKEN=mytoken
docker-compose run myst-libre python example_docker_in_docker.py
```

**Why it works:**
- myst-libre's Authenticator class uses python-dotenv
- python-dotenv's `load_dotenv()` gracefully handles missing .env files
- Environment variables that are already set take precedence over file-based ones
- This approach keeps secrets completely out of the Docker image

### Approach 2: Mount .env File at Runtime

Mount a `.env` file from the host at container startup. This is ideal for:
- Local development
- Complex configurations with multiple settings
- When you prefer file-based configuration

**Defense in Depth: .dockerignore**

The repository includes a [`.dockerignore`](`../.dockerignore`) file that prevents `.env` files from being copied into the image, even if they exist in the build context:

```
.env
.env.local
.env.*.local
.env.secret
```

This provides a safety net: even if someone accidentally runs `docker build` from a directory containing `.env`, the file will **not** be included in the image.

**Verify .env is not in image:**
```bash
# Build the image
docker build -t myst-libre:latest .

# Check that .env is NOT in the image
docker run --rm myst-libre:latest ls -la /workspace/config/.env
# Should output: "cannot access '/workspace/config/.env': No such file or directory"
```

**Option A: Mount single .env file**

**With docker run:**
```bash
docker run -it --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /home/user/workspace:/workspace \
  -v /home/user/.env:/workspace/config/.env:ro \
  -e HOST_WORKSPACE_PATH=/home/user/workspace \
  myst-libre:latest \
  python example_docker_in_docker.py
```

**With docker-compose:**
```yaml
volumes:
  - /var/run/docker.sock:/var/run/docker.sock
  - /home/user/workspace:/workspace
  - /home/user/.env:/workspace/config/.env:ro  # Read-only mount
```

The `:ro` flag makes the mount read-only, preventing accidental modifications.

**Option B: Mount entire config directory**

If you have multiple config files:

**Host structure:**
```
/home/user/myst-config/
├── .env
├── credentials.json
├── config.yaml
└── other-files
```

**With docker-compose:**
```yaml
volumes:
  - /var/run/docker.sock:/var/run/docker.sock
  - /home/user/workspace:/workspace
  - /home/user/myst-config:/workspace/config:ro
```

### Security Checklist

**For Environment Variables (Approach 1):**
- ✓ Secrets passed as `-e VAR=value` flags, never in image
- ✓ Can be managed by secrets manager (HashiCorp Vault, AWS Secrets Manager, etc.)
- ✓ Works seamlessly with CI/CD platforms
- ✓ No files stored on disk
- ✓ Perfect for ephemeral containers

**For .env File (Approach 2):**
- ✓ `.env` file is in `.dockerignore` (prevented from image at build time)
- ✓ `.env` file is mounted at runtime, not baked into image
- ✓ Mount is read-only (`:ro` flag)
- ✓ `.env` file is in `.gitignore` (never in version control)
- ✓ Verify .env is not in image: `docker run --rm myst-libre:latest ls /workspace/config/.env`

**Always:**
- ✓ Credentials never hardcoded in code
- ✓ Rotate credentials regularly
- ✓ Consider using Docker secrets for sensitive data in production
- ✓ Monitor and audit credential access

## Setup

Choose the approach that works best for your use case:

### Approach A: Environment Variables (Recommended for CI/CD)

#### With docker-compose:

1. **Update docker-compose.yml:**

```yaml
services:
  myst-libre:
    image: myst-libre:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /home/user/workspace:/workspace
    environment:
      - HOST_WORKSPACE_PATH=/home/user/workspace
      - DOCKER_PRIVATE_REGISTRY_USERNAME=${DOCKER_PRIVATE_REGISTRY_USERNAME}
      - DOCKER_PRIVATE_REGISTRY_PASSWORD=${DOCKER_PRIVATE_REGISTRY_PASSWORD}
      - CURVENOTE_TOKEN=${CURVENOTE_TOKEN}
```

2. **Set environment variables and run:**

```bash
export DOCKER_PRIVATE_REGISTRY_USERNAME=your_username
export DOCKER_PRIVATE_REGISTRY_PASSWORD=your_password
export CURVENOTE_TOKEN=your_token

docker-compose build
docker-compose run myst-libre python example_docker_in_docker.py
```

#### With manual docker run:

```bash
docker build -t myst-libre:latest .

docker run -it --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /home/user/workspace:/workspace \
  -e HOST_WORKSPACE_PATH=/home/user/workspace \
  -e DOCKER_PRIVATE_REGISTRY_USERNAME=your_username \
  -e DOCKER_PRIVATE_REGISTRY_PASSWORD=your_password \
  -e CURVENOTE_TOKEN=your_token \
  myst-libre:latest \
  python example_docker_in_docker.py
```

### Approach B: Mount .env File (Recommended for Local Development)

#### With docker-compose:

1. **Create your .env file on the host** (outside the repository):

```bash
cat > ~/.env << EOF
DOCKER_PRIVATE_REGISTRY_USERNAME=your_username
DOCKER_PRIVATE_REGISTRY_PASSWORD=your_password
CURVENOTE_TOKEN=your_token
EOF

chmod 600 ~/.env
```

2. **Update docker-compose.yml:**

```yaml
services:
  myst-libre:
    image: myst-libre:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /home/user/workspace:/workspace
      - /home/user/.env:/workspace/config/.env:ro
    environment:
      - HOST_WORKSPACE_PATH=/home/user/workspace
```

3. **Build and run:**

```bash
docker-compose build
docker-compose run myst-libre python example_docker_in_docker.py
```

#### With manual docker run:

```bash
docker build -t myst-libre:latest .

docker run -it --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /home/user/workspace:/workspace \
  -v /home/user/.env:/workspace/config/.env:ro \
  -e HOST_WORKSPACE_PATH=/home/user/workspace \
  myst-libre:latest \
  python example_docker_in_docker.py
```

Replace paths with your actual values:
- `/home/user/workspace` - your workspace directory on host
- `/home/user/.env` - your .env file on host (if using Approach B)

### Approach C: Programmatic (Python)

```python
from myst_libre.tools import JupyterHubLocalSpawner
from myst_libre.rees import REES
from myst_libre.builders import MystBuilder
import os

# Get host path from environment
host_path = os.environ.get('HOST_WORKSPACE_PATH')

rees = REES({...})

hub = JupyterHubLocalSpawner(
    rees,
    host_build_source_parent_dir="/workspace/builds",
    container_build_source_mount_dir='/home/jovyan',
    host_data_parent_dir="/workspace/DATA",
    container_data_mount_dir='/home/jovyan/data',
    # Enable Docker-in-Docker path translation
    host_path_prefix=host_path,
    container_path_prefix="/workspace"
)

hub_logs = hub.spawn_jupyter_hub()
builder = MystBuilder(hub=hub)
myst_logs = builder.build('--execute', '--html')
```

## Configuration

### Required Environment Variables

- **`HOST_WORKSPACE_PATH`**: The absolute path on the host that corresponds to `/workspace` in the container
  - Example: `HOST_WORKSPACE_PATH=/home/user/workspace`
  - Critical for path translation to work correctly

- **`DOCKER_HOST`** (optional): Docker socket path
  - Default: `unix:///var/run/docker.sock`
  - Only needed if using non-standard socket location

### Required Volume Mounts

```yaml
volumes:
  # CRITICAL: Docker socket must be mounted
  - /var/run/docker.sock:/var/run/docker.sock

  # Workspace directory (can be any host path)
  - /home/user/workspace:/workspace
```

### JupyterHubLocalSpawner Parameters

```python
JupyterHubLocalSpawner(
    rees,
    # Required parameters (same as local mode)
    host_build_source_parent_dir="/workspace/builds",
    container_build_source_mount_dir='/home/jovyan',
    host_data_parent_dir="/workspace/DATA",
    container_data_mount_dir='/home/jovyan/data',

    # Docker-in-Docker parameters
    host_path_prefix="/home/user/workspace",        # Host path
    container_path_prefix="/workspace"              # Container path
)
```

## Examples

### Example 1: Basic Usage

See [example_docker_in_docker.py](../example_docker_in_docker.py) for a complete working example.

### Example 2: Running with docker-compose

```bash
# Start an interactive session
docker-compose run myst-libre bash

# Inside the container:
python example_docker_in_docker.py
```

### Example 3: Running a specific repository

```python
import os
from myst_libre.tools import JupyterHubLocalSpawner
from myst_libre.rees import REES
from myst_libre.builders import MystBuilder

rees = REES({
    "registry_url": "https://my-registry.io",
    "gh_user_repo_name": "myuser/myrepo",
    "gh_repo_commit_hash": "main",
    "binder_image_tag": "latest",
    "dotenv": "/workspace/config",
    "bh_project_name": "my-project"
})

hub = JupyterHubLocalSpawner(
    rees,
    host_build_source_parent_dir="/workspace/builds",
    container_build_source_mount_dir='/home/jovyan',
    host_data_parent_dir="/workspace/DATA",
    container_data_mount_dir='/home/jovyan/data',
    host_path_prefix=os.environ['HOST_WORKSPACE_PATH'],
    container_path_prefix="/workspace"
)

hub.spawn_jupyter_hub()
builder = MystBuilder(hub=hub)
builder.build('--execute', '--html', '--pdf')
hub.cleanup()
```

## Troubleshooting

### Issue: Permission Denied on Docker Socket

**Problem:** Error like `permission denied while trying to connect to Docker daemon`

**Solution:**
- Ensure the container runs with a user that has Docker socket access
- The Dockerfile uses root, which has access to the socket
- If using a non-root user, add them to the docker group on the host

### Issue: Volume Mount Path Not Found

**Problem:** Error like `cannot open directory: No such file or directory`

**Solution:**
- Verify `HOST_WORKSPACE_PATH` matches the actual host path
- Check that the path exists on the host: `ls -la /home/user/workspace`
- Ensure you have read/write permissions on the directory

### Issue: Jupyter Container Can't Write to Mounted Volume

**Problem:** Permission denied errors in Jupyter container

**Solution:**
- The host directories should be owned by the host user (not root)
- Make sure volumes are mounted with correct permissions
- The container runs as the host user (via `user=uid:gid`), so permissions should match

### Issue: Jupyter Server Not Accessible

**Problem:** Cannot connect to Jupyter at `http://localhost:XXXX`

**Solution:**
- Verify the port is not already in use: `lsof -i :XXXX`
- Check that the Jupyter container is actually running: `docker ps`
- View container logs: `docker logs <container_id>`
- Check myst-libre container logs for connection errors

### Issue: Path Translation Not Working

**Problem:** Volumes mount to wrong paths

**Verification:**
- Check that `host_path_prefix` is set correctly
- Verify with: `docker inspect <jupyter_container_id>` and check the Mounts section
- If incorrect, check environment variable: `echo $HOST_WORKSPACE_PATH`

## Best Practices

1. **Always use absolute paths**
   - ✓ `host_path_prefix="/home/user/workspace"`
   - ✗ `host_path_prefix="~/workspace"`
   - ✗ `host_path_prefix="./workspace"`

2. **Keep directory structure consistent**
   ```
   /workspace/builds/       ← Repository sources
   /workspace/DATA/         ← External data
   /workspace/config/       ← Configuration files
   ```

3. **Use environment variables**
   - Don't hardcode paths in code
   - Pass via environment variables or config files
   - Makes the setup portable

4. **Monitor volume mounts**
   - Verify with `docker inspect` before and after spawning
   - Check file permissions if experiencing write errors
   - Use `df` inside container to see available space

5. **Handle cleanup properly**
   - Always call `hub.cleanup()` after builds
   - Use context managers when possible:
     ```python
     with JupyterHubLocalSpawner(rees, ...) as hub:
         hub.spawn_jupyter_hub()
         builder = MystBuilder(hub=hub)
         builder.build(...)
     # cleanup() called automatically
     ```

## Differences from Local Mode

| Aspect | Local Mode | Docker-in-Docker |
|--------|-----------|------------------|
| Execution | Host machine | Container |
| Path specification | Absolute host paths | Container paths |
| Path translation | None | Automatic (via `host_path_prefix`) |
| Docker socket | Native access | Mounted volume |
| Cleanup | Manual | Automatic (context manager) |
| Example file | `example2.py` | `example_docker_in_docker.py` |

## Reference

- [Dockerfile](../Dockerfile) - Container image definition
- [docker-compose.yml](../docker-compose.yml) - Docker Compose configuration
- [example_docker_in_docker.py](../example_docker_in_docker.py) - Working example
- [myst_libre/tools/path_utils.py](../myst_libre/tools/path_utils.py) - Path translation utilities

## Further Reading

- [Docker-in-Docker Design Patterns](https://jpetazzo.github.io/2015/09/03/do-not-use-docker-in-docker-for-ci/)
- [Docker Socket Mounting](https://docs.docker.com/engine/reference/commandline/run/#mount-volume--v---read-only)
- [MyST Documentation](https://mystmd.org/)
- [BinderHub Documentation](https://binderhub.readthedocs.io/)
