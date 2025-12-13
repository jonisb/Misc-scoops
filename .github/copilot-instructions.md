# GitHub Copilot Instructions for Misc-scoops

This file provides guidance to GitHub Copilot for working efficiently with this Scoop bucket repository.

## Project Type

This is a **Scoop bucket** - a collection of JSON manifest files that define how to install Windows applications using the [Scoop](https://scoop.sh/) package manager.

## Key Concepts

### Scoop Manifests
- Each `.json` file in `bucket/` is an app manifest following the [Scoop App Manifests specification](https://github.com/ScoopInstaller/Scoop/wiki/App-Manifests)
- Manifests define: version, download URLs, hashes, installation instructions, and auto-update configuration
- Architecture-specific downloads use nested `architecture` objects (`64bit`, `32bit`, `arm64`)

### Required Manifest Fields
When creating or editing manifests, ALWAYS include:
- `version` - Application version string
- `url` OR `architecture` - Download URL(s)
- `hash` - SHA256 hash of the download file(s)

### Common Manifest Fields
- `homepage` - Project homepage URL
- `description` - Brief app description
- `license` - Software license identifier
- `bin` - Executables to add to PATH (can be string or array)
- `shortcuts` - Start menu shortcuts (array of arrays: `[["exe_path", "shortcut_name"]]`)
- `persist` - Files/folders to persist across updates
- `innosetup` - Set to `true` for InnoSetup installers
- `checkver` - Version checking config (often just `"github"` for GitHub releases)
- `autoupdate` - Auto-update URL patterns using `$version` variable

## Manifest URL Patterns

### GitHub Releases
```json
"url": "https://github.com/owner/repo/releases/download/v$version/app-$version.zip"
```

### Architecture-Specific URLs
```json
"architecture": {
    "64bit": {
        "url": "https://example.com/app-x64.zip",
        "hash": "sha256_hash_here"
    },
    "32bit": {
        "url": "https://example.com/app-x86.zip",
        "hash": "sha256_hash_here"
    }
}
```

### URL Fragments
Use `#/dl.exe` or `#/dl.zip` to specify extraction patterns:
```json
"url": "https://example.com/installer.exe#/dl.exe"
```

## Common Tasks

### Adding a New Manifest
1. Create `bucket/app-name.json`
2. Include required fields: `version`, `url`/`architecture`, `hash`
3. Add `checkver` and `autoupdate` for automatic updates
4. Test locally: `scoop bucket add test-bucket .` then `scoop install test-bucket/app-name`

### Updating a Manifest Version
1. Update `version` field
2. Update `url` with new version number (if not using autoupdate)
3. Calculate and update `hash` field(s) with SHA256 of new download
4. The CI workflows will validate on push/PR

### Getting SHA256 Hash
PowerShell:
```powershell
(Get-FileHash -Path "file.exe" -Algorithm SHA256).Hash.ToLower()
```

Bash:
```bash
sha256sum file.exe | awk '{print $1}'
```

### Configuring Auto-Updates
For GitHub releases:
```json
"checkver": "github",
"autoupdate": {
    "url": "https://github.com/owner/repo/releases/download/v$version/app-$version.zip"
}
```

For custom version checking:
```json
"checkver": {
    "url": "https://example.com/releases",
    "regex": "Version ([\\d.]+)"
},
"autoupdate": {
    "url": "https://example.com/download/v$version/app.zip"
}
```

## CI/CD Workflows

### Excavator (`schedule.yml`)
- Runs every 12 hours automatically
- Updates manifests using `checkver` and `autoupdate` configuration
- Uses `ScoopInstaller/GithubActions@main` action
- Commits updates directly to `master` branch

### Test Manifests (`test-manifests.yml`)
- Triggered on push/PR when `bucket/*.json` files change
- Validates JSON syntax and required fields
- Runs `scoop info` to verify manifest correctness
- On PRs: performs full install/uninstall testing
- Requires Windows runner for Scoop testing

### Validate Workflows (`validate-workflows.yml`)
- Triggered when workflow files (`.github/workflows/*.yml`) change
- Uses `actionlint` to lint GitHub Actions syntax
- Runs on Ubuntu (faster than Windows for linting)

### Check URL Freshness (`check-url-freshness.yml`)
- Runs daily at 6 AM UTC (can be manually triggered)
- Checks if download URLs have been modified since manifest was last updated
- Uses `Last-Modified` HTTP headers for detection
- Generates reports in job summary
- Triggers auto-update workflow if stale URLs detected

### Update Stale Manifests (`update-stale-manifests.yml`)
- Manual workflow dispatch (can specify apps to update)
- Uses Scoop's `checkhashes.ps1` to refresh hashes
- Commits changes with `[skip ci]` to avoid triggering tests
- Adds `freshness_notes` field to track refreshes

### Common Workflow Mistakes to Avoid

Based on past fixes (PR #50 and others):

1. **PowerShell Variable + Colon Issue**: When a PowerShell variable is followed by a colon, always use braces
   - ❌ Wrong: `"$app: status"` (PowerShell interprets `$app:` as a provider/drive access like `C:` or `Env:`, causing errors)
   - ✅ Correct: `"${app}: status"` or in regex: `[regex]::Escape("${app}:")`
   - The braces explicitly delimit where the variable name ends

2. **GitHub Actions Expressions**: Use proper syntax for accessing workflow inputs/outputs
   - ✅ Correct: `${{ github.event.inputs.apps }}`
   - ✅ Correct: `${{ needs.job-name.outputs.variable }}`

3. **String Splitting in PowerShell**: Use proper options to handle empty entries
   - ✅ Correct: `"${{ github.event.inputs.apps }}".Split(' ', [System.StringSplitOptions]::RemoveEmptyEntries)`

4. **Exit Code Checking**: Always check `$LASTEXITCODE` correctly
   - ✅ Correct: `if ($LASTEXITCODE -ne 0) { ... }`
   - Native executables set `$LASTEXITCODE`; PowerShell scripts called with `&` may not
   - Always check for null after calling external scripts: `if ($null -eq $LASTEXITCODE) { ... }`
   - Example from update-stale-manifests.yml: `$output = & $checkhashesScript -App $app -Dir $path -Update *>&1` then check if `$LASTEXITCODE` is null

5. **Git Commands in Bash**: Always use `set -euo pipefail` at script start for proper error handling

## PowerShell Script Conventions

### Scoop Installation in Workflows
```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser -Force
iex "& {$(irm get.scoop.sh)}"
echo "$env:USERPROFILE\scoop\shims" | Out-File -FilePath $env:GITHUB_PATH -Encoding utf8 -Append
```

### Testing Manifests Locally
```powershell
# Add local bucket
New-Item -ItemType Junction -Path "$env:USERPROFILE\scoop\buckets\test-bucket" -Target "C:\path\to\repo"

# Install app
scoop install test-bucket/app-name

# Verify
scoop info app-name

# Uninstall
scoop uninstall app-name
```

### Checking Hashes
```powershell
# From repository root
& "$env:USERPROFILE\scoop\apps\scoop\current\bin\checkhashes.ps1" -App app-name -Dir . -Update
```

## Bash Script Conventions

### Safe String Handling
- Use arrays for accumulating items: `declare -a ITEMS=()`
- Add to arrays: `ITEMS+=("value")`
- Join with delimiter: `IFS=,; echo "${ITEMS[*]}"`
- Use null delimiter with jq: `jq -j '... | (. + "\u0000")'`

### Git Operations
```bash
# Get last commit timestamp for file
git log -1 --format="%ct" -- "file.json"

# Get changed files between commits
git diff --name-only "$base_sha" "$head_sha" -- 'bucket/*.json'
```

### HTTP Header Checks
```bash
# Get Last-Modified header
curl -sI -L --max-redirs 5 --max-time 30 -A "User-Agent" "$url" | grep -i "^last-modified:"
```

## JSON Validation Rules

### Valid JSON Structure
- Use 4-space indentation (consistent with existing manifests)
- No trailing commas
- Use double quotes for strings, not single quotes
- Boolean values: `true`/`false` (lowercase)
- Null values: `null` (lowercase)

### Field Ordering (Conventional)
1. `homepage`
2. `description`
3. `license`
4. `version`
5. `url` or `architecture`
6. `hash` (if top-level url)
7. `bin`
8. `shortcuts`
9. `persist`
10. `innosetup`
11. `checkver`
12. `autoupdate`

## Testing Strategy

### Local Testing Before Commit
```powershell
# 1. Validate JSON syntax
Get-Content bucket/app.json | ConvertFrom-Json

# 2. Verify with scoop info
scoop info test-bucket/app

# 3. Test installation
scoop install test-bucket/app

# 4. Verify executables
app --version

# 5. Test uninstall
scoop uninstall app
```

### CI Testing Expectations
- JSON validation happens on all pushes
- Install/uninstall testing ONLY on pull requests (not direct pushes)
- Failed CI on manifests should be fixed before merging

## Common Pitfalls

### Hash Mismatches
- Always recalculate hashes when updating URLs
- Use lowercase for hash values
- Architecture-specific URLs need separate hashes

### URL Fragment Issues
- Don't forget `#/dl.exe` for installers that need renaming
- Fragments are removed when checking HTTP headers

### Version String Consistency
- Ensure `version` matches the actual release version
- Auto-update patterns must match the versioning scheme (with/without 'v' prefix)

### PowerShell String Interpolation
- Use double quotes for variable expansion: `"$var"`
- Avoid unintended expansion in single quotes: `'$var'`
- **CRITICAL**: When a variable is followed by a colon `:`, use braces: `"${var}:"` not `"$var:"` (prevents provider/drive access errors)
- Escape special characters in regex patterns
- Use `$()` for expressions: `"Result: $(Get-Date)"`

### Git Commit Messages
- Use `[skip ci]` suffix to avoid triggering unnecessary workflow runs
- Example: `"Auto-update stale manifests [skip ci]"`

## Best Practices

### Manifest Hygiene
- Always include `homepage`, `description`, and `license`
- Add `checkver` and `autoupdate` for automatic maintenance
- Use meaningful app names (lowercase, hyphens for separators)
- Avoid special characters in filenames

### Workflow Efficiency
- Let Excavator handle routine version updates automatically
- Only manually update manifests when autoupdate cannot work
- Use `workflow_dispatch` for testing workflow changes

### Error Handling
- Check exit codes: `if ($LASTEXITCODE -ne 0)`
- Use try/catch blocks in PowerShell scripts
- Provide meaningful error messages in workflow outputs

### Performance
- Use `fetch-depth: 0` only when full git history is needed
- Cache Scoop installation directory where applicable
- Use Ubuntu runners for non-Scoop operations (faster/cheaper)

## Repository Structure Reference

```
Misc-scoops/
├── bucket/                    # JSON manifest files
│   ├── ollama.json           # Example manifest
│   └── *.json                # App manifests
├── .github/
│   ├── workflows/            # CI/CD automation
│   │   ├── schedule.yml               # Excavator auto-updates
│   │   ├── test-manifests.yml         # Validation & testing
│   │   ├── validate-workflows.yml     # Workflow linting
│   │   ├── check-url-freshness.yml    # URL staleness detection
│   │   └── update-stale-manifests.yml # Manual hash refresh
│   └── copilot-instructions.md  # This file
├── AGENTS.md                 # AI agent guidance
├── LICENSE                   # Unlicense (public domain)
└── README.md                 # Project overview
```

## External Resources

- [Scoop Documentation](https://github.com/ScoopInstaller/Scoop/wiki)
- [App Manifests Specification](https://github.com/ScoopInstaller/Scoop/wiki/App-Manifests)
- [Autoupdate Guide](https://github.com/ScoopInstaller/Scoop/wiki/App-Manifest-Autoupdate)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)

## Development Workflow Summary

1. **Adding/Updating Manifests**: Edit JSON in `bucket/`, ensure required fields, test locally
2. **Committing Changes**: Push to branch, create PR for review
3. **CI Validation**: Automated tests run on PR (JSON validation + install/uninstall)
4. **Merging**: Once CI passes, merge to `master`
5. **Automatic Updates**: Excavator runs every 12 hours to keep manifests current

## License

This project is released under the [Unlicense](http://unlicense.org/) (public domain).
