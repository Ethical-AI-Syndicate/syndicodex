# NPM Publishing Setup for Private Repository

This guide explains how to set up automated NPM publishing for the **private SyndiCodex TypeScript repository**.

## Repository Structure

- **Public Repository**: `Ethical-AI-Syndicate/syndicodex` (documentation, community, examples)
- **Private Repository**: Your private TypeScript codebase (actual CLI implementation)

## NPM Publishing Workflow

The NPM publish CI/CD should be configured in your **private repository** where the actual TypeScript code lives.

### 1. GitHub Actions Workflow

Create `.github/workflows/npm-publish.yml` in your **private repository**:

```yaml
name: Publish to NPM

on:
  release:
    types: [published]
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to publish (leave empty for package.json version)'
        required: false
        type: string

jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
    - name: Checkout repository
      uses: actions/checkout@v4
      with:
        fetch-depth: 0

    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '18'
        registry-url: 'https://registry.npmjs.org'
        cache: 'npm'

    - name: Verify Node.js and npm versions
      run: |
        node --version
        npm --version
        npm whoami
        echo "Publishing as NPM user: $(npm whoami)"

    - name: Install dependencies
      run: npm ci

    - name: Run tests
      run: npm test

    - name: Build TypeScript
      run: npm run build

    - name: Run security audit
      run: npm audit --audit-level=moderate

    - name: Set version from input
      if: github.event.inputs.version != ''
      run: |
        npm version ${{ github.event.inputs.version }} --no-git-tag-version
        echo "PACKAGE_VERSION=${{ github.event.inputs.version }}" >> $GITHUB_ENV

    - name: Get version from package.json
      if: github.event.inputs.version == ''
      run: |
        PACKAGE_VERSION=$(node -p "require('./package.json').version")
        echo "PACKAGE_VERSION=$PACKAGE_VERSION" >> $GITHUB_ENV

    - name: Verify package structure
      run: |
        echo "Publishing version: $PACKAGE_VERSION"
        ls -la dist/
        cat package.json

    - name: Check if version already exists on NPM
      run: |
        if npm view @mcpcodex/syndicodex@$PACKAGE_VERSION version 2>/dev/null; then
          echo "Version $PACKAGE_VERSION already exists on NPM"
          exit 1
        else
          echo "Version $PACKAGE_VERSION is new, proceeding with publish"
        fi

    - name: Dry run npm publish
      run: npm publish --dry-run
      env:
        NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}

    - name: Publish to NPM
      run: npm publish --access public
      env:
        NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}

    - name: Verify published package
      run: |
        sleep 30  # Wait for NPM to propagate
        npm view @mcpcodex/syndicodex@$PACKAGE_VERSION

    - name: Create GitHub release summary
      run: |
        echo "## 🚀 SyndiCodex v$PACKAGE_VERSION Published Successfully!" >> $GITHUB_STEP_SUMMARY
        echo "" >> $GITHUB_STEP_SUMMARY
        echo "### 📦 Package Information" >> $GITHUB_STEP_SUMMARY
        echo "- **Package**: @mcpcodex/syndicodex" >> $GITHUB_STEP_SUMMARY
        echo "- **Version**: $PACKAGE_VERSION" >> $GITHUB_STEP_SUMMARY
        echo "- **Registry**: https://www.npmjs.com/package/@mcpcodex/syndicodex" >> $GITHUB_STEP_SUMMARY
        echo "" >> $GITHUB_STEP_SUMMARY
        echo "### 🛠️ Installation" >> $GITHUB_STEP_SUMMARY
        echo "\`\`\`bash" >> $GITHUB_STEP_SUMMARY
        echo "npm install -g @mcpcodex/syndicodex@$PACKAGE_VERSION" >> $GITHUB_STEP_SUMMARY
        echo "\`\`\`" >> $GITHUB_STEP_SUMMARY
        echo "" >> $GITHUB_STEP_SUMMARY
        echo "### 🔗 Links" >> $GITHUB_STEP_SUMMARY
        echo "- [NPM Package](https://www.npmjs.com/package/@mcpcodex/syndicodex)" >> $GITHUB_STEP_SUMMARY
        echo "- [Documentation](https://github.com/Ethical-AI-Syndicate/syndicodex)" >> $GITHUB_STEP_SUMMARY
```

### 2. Package.json Configuration

Ensure your private repository's `package.json` includes:

```json
{
  "name": "@mcpcodex/syndicodex",
  "version": "1.0.0",
  "description": "🚀 Revolutionary Autonomous Development Intelligence CLI by Ethical AI Syndicate & MCPCodex",
  "main": "./dist/index.js",
  "bin": {
    "syndicodex": "./bin/syndicodex.js",
    "syndx": "./bin/syndicodex.js"
  },
  "files": [
    "dist/**/*",
    "bin/**/*",
    "README.md",
    "LICENSE"
  ],
  "scripts": {
    "build": "tsc",
    "test": "jest",
    "postinstall": "node -e \"console.log('\\n🚀 SyndiCodex installed successfully!\\n\\nNext steps:\\n  1. Run: syndicodex onboarding\\n  2. Visit: https://github.com/Ethical-AI-Syndicate/syndicodex\\n  3. NPM: https://www.npmjs.com/package/@mcpcodex/syndicodex\\n')\""
  },
  "repository": {
    "type": "git",
    "url": "https://github.com/Ethical-AI-Syndicate/syndicodex.git"
  },
  "homepage": "https://github.com/Ethical-AI-Syndicate/syndicodex",
  "preferGlobal": true,
  "publishConfig": {
    "access": "public",
    "registry": "https://registry.npmjs.org/"
  }
}
```

### 3. Required GitHub Secrets

Configure these secrets in your **private repository**:

1. **NPM_TOKEN**: Your NPM access token
   - Go to [npmjs.com](https://www.npmjs.com) → Account Settings → Access Tokens
   - Create a new **Automation** token
   - Add it as `NPM_TOKEN` in GitHub repository secrets

### 4. NPM Organization Setup

Ensure you have access to the `@mcpcodex` organization on NPM:

```bash
# Login as mcpcodexnpm user
npm login

# Verify organization access
npm org ls mcpcodex

# Add yourself to the organization if needed
npm org set mcpcodex mcpcodexnpm developer
```

### 5. Publishing Process

#### Automated (Recommended)
1. Create a GitHub release in your private repository
2. The workflow will automatically build, test, and publish to NPM
3. Package will be available as `@mcpcodex/syndicodex`

#### Manual
```bash
# In your private repository
npm run build
npm test
npm publish --access public
```

### 6. File Structure

Your private repository should include these files for NPM publishing:

```
private-repo/
├── src/                    # TypeScript source
├── dist/                   # Compiled JavaScript (gitignored, built in CI)
├── bin/                    # CLI executable scripts
├── tests/                  # Test files
├── package.json           # NPM configuration
├── tsconfig.json          # TypeScript configuration
├── README.md              # Basic installation instructions
├── LICENSE                # MIT license
└── .github/
    └── workflows/
        └── npm-publish.yml # This CI workflow
```

### 7. Documentation Links

The published NPM package should reference this public documentation repository:

- **Homepage**: `https://github.com/Ethical-AI-Syndicate/syndicodex`
- **Documentation**: Link to this public repo's docs
- **Issues**: Use public repo for community issues and support

This approach keeps your TypeScript source code private while providing comprehensive public documentation and community support.