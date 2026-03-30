# Swiftwave pulling a private GHCR image

This fork can keep its GHCR package private and still be deployed through Swiftwave.

## Target image

Use the fork image instead of the upstream public image:

`ghcr.io/quocdatads4/unla/allinone:stable`

## Current deployment values

- Swiftwave URL: `https://swift.datmarketing.edu.vn`
- Application ID: `7072d1f8-f828-42d5-a6ef-cf264f380503`
- Target image: `ghcr.io/quocdatads4/unla/allinone:stable`
- Public rollback image: `ghcr.io/amoylab/unla/allinone:v0.9.2`
- Expected MCP ingress: `mcp.datmarketing.edu.vn -> 5235 -> https`

## Requirements

- A recent `sw.exe` with these commands:
  - `sw registry list`
  - `sw registry create`
  - `sw registry delete`
  - `sw app update-image`
- Valid Swiftwave authentication on the operator machine
- A GitHub personal access token classic with at least `read:packages`
- The token owner must have access to the private GHCR package

## Create the PAT environment variable

PowerShell:

```powershell
$env:GITHUB_PAT = "<github_pat_classic_with_read_packages>"
```

Do not commit this token into the repository.

## Inspect current state

```powershell
cd C:\Users\QuocDat-PC\Documents\swiftwave\swiftwave_cli
.\sw.exe registry list
.\sw.exe app deployments 7072d1f8-f828-42d5-a6ef-cf264f380503
.\sw.exe app ingress list 7072d1f8-f828-42d5-a6ef-cf264f380503
```

Expected checks:

- `registry list` may be empty before setup
- `app deployments` shows the current image in use
- `app ingress list` should keep `mcp.datmarketing.edu.vn -> 5235 -> https`

## Create a registry credential in Swiftwave

```powershell
cd C:\Users\QuocDat-PC\Documents\swiftwave\swiftwave_cli
.\sw.exe registry create ghcr.io quocdatads4 --password-env GITHUB_PAT
```

Then list credentials:

```powershell
.\sw.exe registry list
```

Example output:

```text
ID   | Username    | URL
-------------------------------
3    | quocdatads4 | ghcr.io
```

Use the returned `ID` in the next step. Do not assume it is always `3`.

## Update the app image

```powershell
.\sw.exe app update-image 7072d1f8-f828-42d5-a6ef-cf264f380503 ghcr.io/quocdatads4/unla/allinone:stable --registry-credential-id <id>
```

Example:

```powershell
.\sw.exe app update-image 7072d1f8-f828-42d5-a6ef-cf264f380503 ghcr.io/quocdatads4/unla/allinone:stable --registry-credential-id 3
```

## Verify the deployment

```powershell
.\sw.exe app deployments 7072d1f8-f828-42d5-a6ef-cf264f380503
.\sw.exe app ingress list 7072d1f8-f828-42d5-a6ef-cf264f380503
```

Expected result:

- latest deployment uses `ghcr.io/quocdatads4/unla/allinone:stable`
- latest deployment status is `deployed`
- ingress still points `mcp.datmarketing.edu.vn` to port `5235` over `https`

## Full copy/paste sequence

```powershell
$env:GITHUB_PAT = "<github_pat_classic_with_read_packages>"

cd C:\Users\QuocDat-PC\Documents\swiftwave\swiftwave_cli

.\sw.exe registry list
.\sw.exe registry create ghcr.io quocdatads4 --password-env GITHUB_PAT
.\sw.exe registry list
.\sw.exe app update-image 7072d1f8-f828-42d5-a6ef-cf264f380503 ghcr.io/quocdatads4/unla/allinone:stable --registry-credential-id <id>
.\sw.exe app deployments 7072d1f8-f828-42d5-a6ef-cf264f380503
.\sw.exe app ingress list 7072d1f8-f828-42d5-a6ef-cf264f380503
```

Replace:

- `<github_pat_classic_with_read_packages>`
- `<id>` from `sw registry list`

## Rollback

Rollback to the current public reference image:

```powershell
.\sw.exe app update-image 7072d1f8-f828-42d5-a6ef-cf264f380503 ghcr.io/amoylab/unla/allinone:v0.9.2 --clear-registry-credential
.\sw.exe app deployments 7072d1f8-f828-42d5-a6ef-cf264f380503
```

Rollback to an older private tag:

```powershell
.\sw.exe app update-image 7072d1f8-f828-42d5-a6ef-cf264f380503 ghcr.io/quocdatads4/unla/allinone:v1.2.3 --registry-credential-id <id>
.\sw.exe app deployments 7072d1f8-f828-42d5-a6ef-cf264f380503
```

## Troubleshooting

### `denied` while pulling the image

Common causes:

- the package is private and no registry credential is attached
- the PAT does not include `read:packages`
- the PAT owner does not have access to the package
- the wrong credential ID was used
- the app still points at an outdated credential

Fix sequence:

1. Run `sw registry list`
2. Confirm the `ghcr.io` credential exists for `quocdatads4`
3. Recreate the PAT classic with `read:packages`
4. Recreate the credential if needed
5. Run `sw app update-image ... --registry-credential-id <id>` again
6. Recheck `sw app deployments ...`

### Cleaning up old credentials

List credentials:

```powershell
.\sw.exe registry list
```

Delete an unused credential:

```powershell
.\sw.exe registry delete 3
```

Only remove a credential after confirming the app no longer depends on it.
