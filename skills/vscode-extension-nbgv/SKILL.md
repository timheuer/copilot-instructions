---
name: vscode-extension-nbgv
description: This skill helps ensure that a new VS Code Extension is configured correctly in the metadata and establishes and automatic git-based versioning.
---
You are going to help make changes to the package.json for a VS Code extension and also setup the GitHub Actions CI build workflow appropriately using automatic versioning with Nerdbank Git Versioning (NBGV).

Use the #askQuestions tool to ask any clarifying questions you need to ensure you can complete the task or for missing information.

# Modifying package.json

The following needs to be ensured in the package.json file ONLY IF NOT SPECIFIED:

- version is: 0.0.0-placeholder
- author is needs to have a name and a url (ask if not known)
- publisher is the same as the Marketplace publisher name (ask if not known)
- icon is "icon.png" (validate this file exists and is at the root, summarize if it is not)
- repository is type 'git' with the URL of the remote repository of the workspace
- bugs property should be the /issues of the repository
- qna property should be the /issues of the repository
- in any workflow step that uses the Marketplace ID get that from the package.json

# Ensuring scripts in package.json

These scripts should be defined in the package.json

- deploy: which is a 'vsce publish'

# Build workflow

The build workflow should build the package, upload the artiface and publish to the marketplace using Nerdbank Git Versioning like so:

Example workflow:

```
name: Build and Publish

on:
  workflow_dispatch:
    branches:
      - main
    paths-ignore:
      - '**/*.md'
      - '**/*.gitignore'
      - '**/*.gitattributes'
  push:
    branches:
      - main
    paths-ignore:
      - '**/*.md'
      - '**/*.gitignore'
      - '**/*.gitattributes'

jobs:
  build:
    permissions:
      contents: write
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0

    - uses: actions/setup-node@v4
      with:
        node-version: 20

    - name: 📦 Install dependencies
      run: npm install

    - name: 🎁 Install vsce
      run: npm install -g @vscode/vsce

    - name: 🖥️ Install cross-env
      run: npm install -g cross-env

    - name: 🏷️ NBGV
      uses: dotnet/nbgv@master
      id: nbgv
      with:
        stamp: package.json

    - name: 🗣️ NBGV outputs
      run: |
        echo "SimpleVersion: ${{ steps.nbgv.outputs.SimpleVersion }}"

    - name: 📦 Package
      run: |
        vsce package -o ./${{ github.event.repository.name}}.vsix

    - name: 📰 Publish
      if: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
      run: npm run deploy
      env:
        VSCE_PAT: ${{ secrets.VSCE_PAT }}

    - name: 🔗 Marketplace Link
      if: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
      shell: pwsh
      run: |
        Write-Host "::notice::Published to VS Code Marketplace: https://marketplace.visualstudio.com/items?itemName=marketplace-id"

    - name: ⬆️ Upload artifact
      uses: actions/upload-artifact@v4
      with:
        name: ${{ github.event.repository.name}}.vsix
        path: ./*.vsix

    - name: 🏷️ Tag and Release
      id: tag_release
      uses: softprops/action-gh-release@v1
      with:
        body: Release ${{ steps.nbgv.outputs.SimpleVersion }}
        tag_name: ${{ steps.nbgv.outputs.SimpleVersion }}
        generate_release_notes: true
        files: ./*.vsix
```

When ensuring there is a `version.json` file for NBGV it should look like this (default to 0.1 version):

```
{
    "$schema": "https://raw.githubusercontent.com/dotnet/Nerdbank.GitVersioning/main/src/NerdBank.GitVersioning/version.schema.json",
    "version": "0.1",
    "publicReleaseRefSpec": [
        "^refs/heads/main$", 
        "^refs/tags/v\\d+\\.\\d+"
    ],
    "cloudBuild": {
        "buildNumber": {
            "enabled": true
        }
    }
  }
```

When summarizing be sure to remind to set `VSCE_PAT` using a marketplace PAT or the workflow will not work.

Also verify the extension output using `vsce ls` to ensure that only the intended files are included in the package. Suggest any changes to the `.vscodeignore` file as needed.

# License File

Be sure if `LICENSE.md` or `LICENSE` does not exist to create one and make it the standard MIT license.