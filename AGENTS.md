# AGENTS.md

This file provides guidance for AI agents and automated tools working on this repository.

## Project Overview

This repository is a **Scoop bucket** - a collection of JSON manifest files that define how to install Windows applications using the [Scoop](https://scoop.sh/) package manager.

### Repository Structure

```
Misc-scoops/
├── bucket/           # Contains JSON manifest files for each application
│   └── *.json        # App manifests (e.g., ollama.json, Magpie.json)
├── .github/
│   └── workflows/    # CI/CD workflows
│       ├── schedule.yml          # Auto-update via Excavator
│       ├── test-manifests.yml    # Manifest validation and testing
│       └── validate-workflows.yml # Workflow file validation
├── LICENSE           # Unlicense (public domain)
└── README.md         # Project readme
```

## Working with Manifests

### Manifest File Format

Each JSON file in the `bucket/` directory is a Scoop manifest following the [Scoop App Manifests specification](https://github.com/ScoopInstaller/Scoop/wiki/App-Manifests).

Required fields:
- `version` - The current version of the application
- `url` or `architecture` - Either a top-level `url` field for single downloads, or an `architecture` object containing nested URLs for different architectures (e.g., `64bit`, `32bit`)

Common fields:
- `homepage` - Project homepage URL
- `description` - Brief description of the application
- `license` - Software license
- `hash` - SHA256 hash of the download
- `bin` - Executable(s) to add to PATH
- `shortcuts` - Start menu shortcuts to create
- `checkver` - Version checking configuration for auto-updates
- `autoupdate` - Auto-update URL patterns

### Adding or Updating Manifests

1. **Adding a new app**: Create a new JSON file in `bucket/` with the app name
2. **Updating versions**: Update the `version` field and `hash` for new releases
3. **Auto-updates**: Configure `checkver` and `autoupdate` for automatic version detection

### Validation

Manifests are validated by CI workflows on pull requests:
- JSON syntax validation
- Required fields check
- `scoop info` verification
- Install/uninstall testing (on PRs)

## CI/CD Workflows

### Excavator (schedule.yml)
Runs every 12 hours to automatically update manifest versions using the `checkver` and `autoupdate` configurations.

### Test Manifests (test-manifests.yml)
Validates changed manifests on pushes and PRs:
- Validates JSON syntax and required fields
- Tests install/uninstall on pull requests

### Validate Workflows (validate-workflows.yml)
Lints GitHub Actions workflow files using actionlint.

## Development Guidelines

### When Modifying Manifests
- Ensure valid JSON syntax
- Include all required fields (`version`, `url`/`architecture`)
- Provide accurate SHA256 hashes for downloads
- Test locally by adding the bucket with `scoop bucket add <name> <path>` and then `scoop install <name>/<app>`

### When Modifying Workflows
- Ensure YAML syntax is valid
- Test workflow logic changes carefully
- Workflows are linted with actionlint on PR

## License

This project is released under the [Unlicense](http://unlicense.org/) (public domain).
