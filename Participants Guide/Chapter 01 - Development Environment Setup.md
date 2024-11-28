# Cloud-Native SaaS Application Development Workshop
## Participant Guide

### Table of Contents
1. Development Environment Setup
2. Understanding Cloud-Native Architecture
3. Building Your First Serverless Application
4. Frontend Development with React
5. Backend Development with Python
6. Database Integration
7. AI Integration
8. Testing and Deployment
9. Scaling and Monitoring
10. Best Practices and Production Readiness

# Chapter 1: Development Environment Setup

## Introduction
This chapter guides you through setting up all necessary tools and development environments for both Windows and macOS users. Whether you're a beginner or experienced developer, following these steps will ensure you have everything needed to build cloud-native applications.

## Prerequisites
- Computer running Windows 10/11 or macOS 10.15+
- Stable internet connection
- At least 8GB RAM (16GB recommended)
- 20GB free disk space

## Essential Tools Installation Guide

### 1. Code Editor
**Visual Studio Code** (Recommended for beginners)
- Windows/macOS: Download from https://code.visualstudio.com/
- Essential Extensions:
  - Python
  - JavaScript and React
  - Live Server
  - GitLens
  - Thunder Client (API testing)

Installation steps:
1. Download the appropriate version for your OS
2. Run the installer
3. Launch VS Code
4. Open Extensions panel (Ctrl+Shift+X or Cmd+Shift+X)
5. Install recommended extensions

### 2. Version Control
**Git**
- Windows: Download from https://git-scm.com/download/win
- macOS: 
  ```bash
  /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
  brew install git
  ```

### 3. Programming Languages

#### Python Setup
- Windows: Download from https://www.python.org/downloads/ (Choose Python 3.11+)
- macOS:
  ```bash
  brew install python
  ```

Verify installation:
```bash
python --version
pip --version
```

#### Node.js Setup (for React development)
- Windows/macOS: Download LTS version from https://nodejs.org/
- macOS (alternative):
  ```bash
  brew install node
  ```

Verify installation:
```bash
node --version
npm --version
```

### 4. Cloud Provider CLI Tools

#### AWS CLI
- Windows: Download AWS CLI MSI installer
- macOS:
  ```bash
  curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
  sudo installer -pkg AWSCLIV2.pkg -target /
  ```

### 5. Database Tools
**MongoDB Compass** (GUI for MongoDB)
- Download from https://www.mongodb.com/try/download/compass

### 6. API Development Tools
**Postman**
- Download from https://www.postman.com/downloads/

### 7. Terminal Setup

#### Windows Users
- Windows Terminal (recommended)
  - Install from Microsoft Store
- Git Bash (comes with Git installation)

#### macOS Users
- Terminal (pre-installed)
- iTerm2 (recommended)
  ```bash
  brew install --cask iterm2
  ```

## Verification Checklist

Use this checklist to ensure all tools are properly installed:

```bash
# Run these commands in your terminal

# 1. Git
git --version

# 2. Python
python --version
pip --version

# 3. Node.js
node --version
npm --version

# 4. AWS CLI
aws --version

# 5. Create a test Python virtual environment
python -m venv test-env
# Windows
test-env\Scripts\activate
# macOS
source test-env/bin/activate
```

## Common Issues and Solutions

### Windows-Specific Issues
1. **Python not found in PATH**
   - Solution: Reinstall Python, ensure "Add to PATH" is checked
   
2. **Permission errors**
   - Solution: Run terminal as Administrator

### macOS-Specific Issues
1. **Command Line Tools missing**
   - Solution: Run `xcode-select --install`
   
2. **Homebrew installation fails**
   - Solution: Check System Integrity Protection settings

## Next Steps
Once you've completed the setup:
1. Test each tool to ensure proper functionality
2. Join our workshop Slack channel for support
3. Review Chapter 2 for understanding cloud-native architecture

## Support Resources
- Workshop Slack Channel: [Link]
- Technical Documentation Repository: [Link]
- Stack Overflow: https://stackoverflow.com/
- Official tool documentation links

Need help? Contact our workshop support team at [email/contact info].
