---
name: nuget-trusted-publishing
description: >-
  Migrate, configure, or troubleshoot keyless Trusted Publishing (OIDC) for NuGet packages using
  GitHub Actions and nuget.org without static API keys. Use when setting up publishing workflows,
  migrating from NUGET_API_KEY to NuGet/login@v1, or fixing 403 untrusted token errors.
---

# NuGet Trusted Publishing Migration & Configuration Skill

This skill guides you through migrating .NET repositories from legacy, static NuGet API keys (`NUGET_API_KEY`) to keyless **Trusted Publishing** (OpenID Connect / OIDC) between GitHub Actions and [nuget.org](https://www.nuget.org).

---

## When to Use This Skill

- Migrating an existing GitHub Actions workflow (e.g., `publish.yml` or `release.yml`) from `-k ${{ secrets.NUGET_API_KEY }}` to OIDC.
- Setting up a new NuGet publishing workflow for a .NET project.
- Troubleshooting `403 Forbidden` / `Untrusted token` or `Resource not accessible by integration` errors.

---

## Migration Overview

| Component | Legacy (Static Key) | Trusted Publishing (OIDC) |
| :--- | :--- | :--- |
| **Authentication** | Static `NUGET_API_KEY` in GitHub Secrets | Short-lived OIDC token exchanged via `NuGet/login@v1` |
| **Credential Lifetime** | Up to 365 days (manual rotation required) | Single-use, valid for ~1 hour |
| **Job Permissions** | Default (read) | `id-token: write` |
| **Required Secret** | `NUGET_API_KEY` (secret) | `NUGET_USER` (secret or variable containing NuGet username) |
| **Server Policy** | Manual API key creation on NuGet.org | Trusted Publisher policy on NuGet.org mapped to repo & workflow |

---

## Step 1: Audit Repository & Existing Workflow

Run the following checks to gather repository metadata:

1. **Identify GitHub Owner and Repository Name:**
   ```powershell
   git remote get-url origin
   ```
   *Example: `https://github.com/trippntechnology/tnt.commons.git` &rarr; Owner: `trippntechnology`, Repository: `tnt.commons`.*

2. **Locate the Publish Workflow:**
   Search `.github/workflows/` for existing publish steps:
   ```powershell
   Get-ChildItem -Path .github/workflows/*.yml | Select-String -Pattern "dotnet nuget push|NUGET_API_KEY"
   ```
   Note the exact workflow **filename** (e.g. `publish.yml`).

3. **Verify Project Name & Directory Casing:**
   Check the exact casing of the project folder producing the `.nupkg`:
   ```powershell
   Get-ChildItem -Filter *.csproj -Recurse | Select-Object FullName
   ```
   *Example: `TNT.Commons\bin\Release\*.nupkg` (ensure casing matches the filesystem).*

---

## Step 2: Configure Policy on NuGet.org (Web Portal)

> [!NOTE]
> NuGet.org does not provide a CLI or administrative API to automate policy creation; this step must be completed in the browser.

1. Open [nuget.org/account/trustedpublishing](https://www.nuget.org/account/trustedpublishing) in your browser.
   *(Alternatively, for an existing package, navigate to **Manage Packages** &rarr; select package &rarr; **Trusted Publishers**).*
2. Click **Create** or **Add trusted publisher** and choose **GitHub Actions**.
3. Fill in the policy details using the values determined in Step 1:
   - **Policy Name**: `<repo>-publish` (e.g., `tnt.commons-publish`).
   - **Package Owner**: Select your NuGet username or organization (e.g., `trippntechnology`).
   - **Repository Owner**: The GitHub organization or user (e.g., `trippntechnology`).
   - **Repository**: The repository name (e.g., `tnt.commons`).
   - **Workflow File**: **Filename only** without any path (e.g., `publish.yml` &mdash; **do NOT** enter `.github/workflows/publish.yml`).
   - **Environment**: Leave blank unless your GitHub job specifically targets an Environment.
4. Click **Save** / **Create policy**.

---

## Step 3: Ensure GitHub Has `NUGET_USER`

Make sure your NuGet account/organization username is accessible to the GitHub workflow:

- **Via GitHub Web UI:** Go to repository or organization **Settings** &rarr; **Secrets and variables** &rarr; **Actions** &rarr; create a Secret or Variable named `NUGET_USER` with your NuGet account name (e.g. `trippntechnology`).
- **Via GitHub CLI (Optional):**
  ```powershell
  gh secret set NUGET_USER --body "trippntechnology"
  # Or as a repository variable:
  gh variable set NUGET_USER --body "trippntechnology"
  ```

> [!IMPORTANT]
> `NUGET_USER` must be your NuGet username/organization handle, **not** an email address.

---

## Step 4: Update GitHub Actions Workflow YAML

Edit `.github/workflows/<workflow-file>.yml` (e.g. `publish.yml`):

### 1. Grant OIDC Token Permissions to the Job
Add `permissions:` with `id-token: write` directly under the job definition:
```yaml
jobs:
  build:
    runs-on: windows-latest # or ubuntu-latest
    permissions:
      id-token: write  # Required for OIDC token issuance
```

### 2. Insert `NuGet/login@v1` Before Publishing
Insert the login step between the build step and the publish step:
```yaml
    - name: NuGet login (OIDC → temp API key)
      uses: NuGet/login@v1
      id: login
      with:
        user: ${{ secrets.NUGET_USER }}
```

### 3. Update `dotnet nuget push`
Change the API key argument to consume `${{ steps.login.outputs.NUGET_API_KEY }}`:
```yaml
    - name: Publish
      run: dotnet nuget push "<ProjectDir>\bin\Release\*.nupkg" --api-key ${{ steps.login.outputs.NUGET_API_KEY }} --source https://api.nuget.org/v3/index.json --skip-duplicate
```

### Example Diff (from TNT.Commons `7ea739d`):
```diff
 jobs:
   build:
     runs-on: windows-latest
+    permissions:
+      id-token: write  # Required for OIDC token issuance
 
     steps:
     - uses: actions/checkout@v4
     - name: Setup .NET
       uses: actions/setup-dotnet@v4
       with:
         dotnet-version: '10.0.x'
     - name: Restore dependencies
       run: dotnet restore
     - name: Build
       run: dotnet build --configuration Release --no-restore
+    - name: NuGet login (OIDC → temp API key)
+      uses: NuGet/login@v1
+      id: login
+      with:
+        user: ${{ secrets.NUGET_USER }}
     - name: Publish
-      run: dotnet nuget push "TNT.Commons\bin\Release\*.nupkg" -k ${{ secrets.NUGET_API_KEY }} -s "https://api.nuget.org/v3/index.json" -n --skip-duplicate
+      run: dotnet nuget push "TNT.Commons\bin\Release\*.nupkg" --api-key ${{ steps.login.outputs.NUGET_API_KEY }} --source https://api.nuget.org/v3/index.json --skip-duplicate
```

---

## Step 5: Verify and Clean Up

1. Commit and push the workflow changes to your release branch (or trigger a test run).
2. Check the GitHub Actions run logs:
   - Ensure the `NuGet/login@v1` step succeeds and logs that an OIDC token was acquired.
   - Ensure `dotnet nuget push` successfully uploads the package.
3. Once verified, delete the old `NUGET_API_KEY` from repository secrets to eliminate unused long-lived credentials:
   ```powershell
   gh secret delete NUGET_API_KEY
   ```

---

## Troubleshooting & Common Pitfalls

### 1. `403 Forbidden` / `Untrusted token` / `No matching policy found`
- **Workflow File Path Error**: Ensure the NuGet.org policy has **only** the filename (e.g. `publish.yml`), NOT `.github/workflows/publish.yml`.
- **Casing & Typo Mismatches**: Check that `Repository Owner` and `Repository` in the policy match your GitHub URL exactly.
- **Incorrect `NUGET_USER`**: Confirm that `NUGET_USER` matches the account/org owning the package and is not an email address.

### 2. `Resource not accessible by integration`
- You forgot to declare `permissions: id-token: write` on the job.

### 3. File Not Found During Push (`*.nupkg does not exist`)
- Verify path casing on Windows vs Linux (e.g. `TNT.Commons` vs `tnt.commons`).
- Ensure the project is built with `--configuration Release` or `dotnet pack` before the push step.

---

## Additional References

- [Architecture & Mechanics Reference](file:///D:/Src/tnt.agentplugins/plugins/tnt-nuget/skills/nuget-trusted-publishing/references/guide.md)
- [Gold Standard publish.yml Template](file:///D:/Src/tnt.agentplugins/plugins/tnt-nuget/skills/nuget-trusted-publishing/examples/publish.yml)
- [Official Microsoft Documentation](https://learn.microsoft.com/en-us/nuget/nuget-org/trusted-publishing)
