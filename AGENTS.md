# AGENTS.md

This file provides guidance to agents when working with code in this repository.

## Project Overview

This repository contains an **Open Horizon edge service** that deploys and manages a containerized Minecraft Java Edition server on edge nodes. The service leverages Open Horizon's policy-based deployment system to automatically distribute and run Minecraft servers across distributed edge infrastructure.

**Key Technologies:**
- **Open Horizon**: Edge computing platform for autonomous service deployment and management
- **Docker**: Container runtime using the third-party `itzg/minecraft-server` image
- **Minecraft Server**: Java-based game server (supports vanilla, Spigot, Forge, and other variants)

**Architecture:**
- Uses a pre-built Docker image (`itzg/minecraft-server`) rather than building custom images
- Deploys via Open Horizon's policy-based agreement system
- Supports multi-architecture deployment (amd64, arm64)
- Persistent data storage via Docker volumes

## Repository Structure

```
service-minecraft/
├── horizon/                    # Open Horizon configuration files
│   ├── service.definition.json # Service metadata and deployment config
│   ├── service.policy.json     # Service-side policy constraints
│   ├── deployment.policy.json  # Hub-side deployment policy
│   ├── node.policy.json        # Edge node policy for registration
│   └── userinput.json          # User input configuration
├── Makefile                    # Build, test, and deployment automation
├── README.md                   # User-facing documentation
└── LICENSE.txt                 # License information
```

## Configuration and Customization

### Environment Variables

The service is configured through environment variables defined in the Makefile:

**Open Horizon Configuration:**
- `HZN_ORG_ID`: Organization ID in the Open Horizon hub (default: `examples`)
- `SERVICE_ORG_ID`: Service organization (defaults to `HZN_ORG_ID`)
- `SERVICE_NAME`: Service identifier (default: `service-minecraft`)
- `SERVICE_VERSION`: Service version (default: `0.0.2`)

**Minecraft Server Configuration:**
- `MINECRAFT_SERVER_PORT`: Game server port (default: `25565`)
- `MINECRAFT_RCON_PORT`: Remote console port (default: `25575`)
- `MEMORY`: Server memory allocation (default: `2G`)
- `TYPE`: Server type - SPIGOT, VANILLA, FORGE, etc. (default: `SPIGOT`)
- `VERSION`: Minecraft version (default: `1.19`)
- `EULA`: Auto-accept Minecraft EULA (default: `TRUE`)

### Policy-Based Deployment

The service uses Open Horizon's policy system with three policy files:

1. **service.policy.json**: Defines service properties and constraints
   - Property: `prop1 = value1`
   - Constraint: `minecraft == server`

2. **deployment.policy.json**: Hub-side policy for automatic deployment
   - Matches nodes with `minecraft == server` property
   - Supports any architecture (`arch: "*"`)

3. **node.policy.json**: Edge node registration policy
   - Sets node property `minecraft = server` to match deployment policy

## Development Workflow

### Prerequisites

- Open Horizon Management Hub access (or temporary developer account)
- Open Horizon agent (`anax`) installed on edge node
- Docker installed and running
- Minecraft Java Edition client (for testing)
- Optional utilities: `gcc`, `make`, `git`, `jq`, `curl`, `net-tools`

### Common Commands

**Local Testing:**
```bash
make run          # Run Minecraft server locally in Docker
make attach       # Connect to running container shell
make test         # Test server responsiveness (returns Java error - expected)
make stop         # Stop local container
```

**Service Publishing and Deployment:**
```bash
make publish      # Complete workflow: publish service + policies + register node
make agent-run    # Register node with hub using node policy
make agent-stop   # Unregister node and stop all services
```

**Individual Publishing Steps:**
```bash
make publish-service              # Publish service definition to hub
make publish-service-policy       # Publish service policy
make publish-deployment-policy    # Publish deployment policy
```

**Debugging and Monitoring:**
```bash
make log                # View event logs and service logs
make deploy-check       # Verify policy compatibility
```

**Cleanup:**
```bash
make clean              # Remove local Docker image and volume
make distclean          # Full cleanup: unregister + remove policies + clean
```

## Important Development Notes

### Service Definition Structure

The `horizon/service.definition.json` file defines:
- **User inputs**: Configurable parameters (MEMORY, EULA, TYPE, VERSION)
- **Deployment**: Container image, ports, and volume bindings
- **Sharable**: Set to "multiple" - allows multiple instances per node

### Docker Image Strategy

This service **does not build custom images**. It uses the community-maintained `itzg/minecraft-server` image:
- `make build` and `make push` are no-ops
- All customization happens through environment variables
- Refer to [itzg/docker-minecraft-server documentation](https://github.com/itzg/docker-minecraft-server) for advanced configuration

### Testing Considerations

- Server startup takes several minutes (world generation)
- Testing requires Minecraft Java Edition client
- Connect to `localhost:25565` (or configured port)
- Use `make attach` to access server console and run commands
- Stop server gracefully with `rcon-cli stop` command

### Policy Matching

For successful deployment, ensure:
1. Node policy sets `minecraft = server` property
2. Service policy requires `minecraft == server` constraint
3. Deployment policy requires `minecraft == server` constraint
4. All three policies must align for agreement formation

### Volume Persistence

- Minecraft world data persists in Docker volume `minecraft-data`
- Volume survives container restarts
- Use `make clean` to remove volume and reset world

## Conventions and Best Practices

1. **Always export required environment variables** before running make targets
2. **Use `make deploy-check`** before publishing to verify policy compatibility
3. **Monitor agreement formation** with `hzn agreement list` after registration
4. **Check logs** if deployment fails: `make log` provides detailed diagnostics
5. **Graceful shutdown**: Always use `rcon-cli stop` before removing containers
6. **Version management**: Update `SERVICE_VERSION` when making changes to service definition
7. **README.md quality**: Always lint and spellcheck the README.md file before committing and pushing changes to ensure documentation quality and consistency

### Automated Documentation Quality Checks

The repository includes automated GitHub Actions workflows that run on every commit affecting README.md:

- **Markdown Linting**: Uses `markdownlint-cli` with configuration in `.markdownlint.json`
- **Spellchecking**: Uses `cspell` with custom dictionary in `.cspell.json`
- **Workflow File**: `.github/workflows/documentation-quality.yml`

The workflow status is displayed via a badge at the top of README.md. Both checks must pass before merging changes.

**Configuration Files:**
- `.markdownlint.json`: Markdown linting rules (allows line length up to 120 chars, permits HTML img tags)
- `.cspell.json`: Custom dictionary for technical terms (anax, containerized, Spigot, etc.)

## Troubleshooting

**Agent not configured:**
- Run `hzn node list` to check configuration state
- Should show "unconfigured" before registration
- Verify `exchange_api` and `mms_api` URLs are set

**Agreement not forming:**
- Run `make deploy-check` to verify policy compatibility
- Check that node, service, and deployment policies all match
- Verify node has required architecture (amd64 or arm64)

**Server not starting:**
- Check container logs: `docker logs service-minecraft`
- Verify EULA is set to TRUE
- Ensure sufficient memory allocation (minimum 1G recommended)
- Confirm ports 25565 and 25575 are not in use

**Cannot connect to server:**
- Wait 3-5 minutes for initial world generation
- Verify server is running: `docker ps | grep minecraft`
- Check port mapping: `docker port service-minecraft`
- Test connectivity: `nc -zv localhost 25565`
