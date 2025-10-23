# Git Setup Guide

This guide helps you set up version control for your interview preparation knowledge base.

## 🎯 Benefits of Using Git

- ✅ **Version History**: Track all changes to your notes
- ✅ **Backup**: Your notes are safely stored on GitHub
- ✅ **Access Anywhere**: Clone and access from any computer
- ✅ **Sync Across Devices**: Keep notes synchronized
- ✅ **Undo Changes**: Revert mistakes easily

## 📋 Initial Setup

### 1. Install Git
If you don't have Git installed:
- Windows: Download from https://git-scm.com/download/win
- Mac: `brew install git` or download from https://git-scm.com/download/mac
- Linux: `sudo apt-get install git` (Ubuntu/Debian)

### 2. Configure Git
```bash
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
```

### 3. Initialize Repository
Navigate to your interview-prep folder and run:
```bash
cd /path/to/interview-prep
git init
```

### 4. Create .gitignore
Create a `.gitignore` file to exclude unnecessary files:
```
# OS files
.DS_Store
Thumbs.db

# Editor files
.vscode/
.idea/

# Temporary files
*.tmp
*.bak
~*

# Personal notes you don't want to sync
/personal-notes/
```

### 5. First Commit
```bash
git add .
git commit -m "Initial commit: Interview prep knowledge base"
```

## 🔄 Daily Workflow

### Adding New Content
```bash
# Check what files changed
git status

# Stage your changes
git add .

# Or stage specific files
git add csharp/new-topic.md

# Commit with a descriptive message
git commit -m "Add notes on Entity Framework Core"

# Push to GitHub (after setting up remote)
git push origin main
```

### Viewing History
```bash
# See commit history
git log

# See what changed in a specific commit
git show <commit-hash>

# See changes in a file
git log -p csharp/async-await.md
```

### Undoing Changes
```bash
# Discard changes to a file (not yet staged)
git checkout -- filename.md

# Unstage a file (still keep changes)
git reset HEAD filename.md

# Revert to a previous commit
git revert <commit-hash>
```

## 🌐 Setting Up GitHub Remote

### 1. Create GitHub Repository
1. Go to https://github.com and sign in
2. Click "New Repository"
3. Name it: `interview-prep` or `cs-interview-notes`
4. Make it **Private** (your personal notes)
5. Don't initialize with README (we already have content)
6. Click "Create Repository"

### 2. Link Local Repository to GitHub
```bash
# Add remote (replace USERNAME with your GitHub username)
git remote add origin https://github.com/USERNAME/interview-prep.git

# Verify remote
git remote -v

# Push to GitHub
git branch -M main
git push -u origin main
```

### 3. Subsequent Pushes
After initial setup, just use:
```bash
git push
```

## 📝 Recommended Commit Messages

Use clear, descriptive commit messages:

```bash
# Good commit messages
git commit -m "Add async/await examples and best practices"
git commit -m "Update SOLID principles with LSP examples"
git commit -m "Add common SQL optimization techniques"
git commit -m "Fix code example in DI section"
git commit -m "Add interview questions for Entity Framework"

# Poor commit messages (avoid these)
git commit -m "update"
git commit -m "changes"
git commit -m "stuff"
```

## 🔍 Useful Git Commands

```bash
# See what changed (before committing)
git diff

# See changes in staged files
git diff --staged

# See all branches
git branch

# Create a new branch (for experimental notes)
git checkout -b experimental-topics

# Switch back to main branch
git checkout main

# Merge changes from another branch
git merge experimental-topics

# Pull latest changes from GitHub
git pull

# Clone repository on another computer
git clone https://github.com/USERNAME/interview-prep.git
```

## 🎯 Suggested Workflow

### Daily Review Session
```bash
# Start of day - pull any changes from other devices
git pull

# Make updates to notes during study session
# [edit files in VS Code]

# End of day - commit your progress
git add .
git commit -m "Daily review: Updated async/await and added DI examples"
git push
```

### Before an Interview
```bash
# Quick review of recent topics
git log --since="1 week ago" --oneline

# Create a backup branch
git checkout -b backup-before-interview

# Go back to main
git checkout main
```

## 🔐 Security Note

- Keep your repository **private** (contains your personal notes)
- Never commit sensitive information (passwords, API keys, company secrets)
- Your GitHub account should have two-factor authentication enabled

## 📚 Learning More About Git

- Official Git Documentation: https://git-scm.com/doc
- GitHub Guides: https://guides.github.com/
- Interactive Git Tutorial: https://learngitbranching.js.org/

---

## Quick Start Checklist

- [ ] Install Git
- [ ] Configure user name and email
- [ ] Initialize repository (`git init`)
- [ ] Create .gitignore file
- [ ] Make first commit
- [ ] Create private GitHub repository
- [ ] Link local repo to GitHub
- [ ] Push to GitHub
- [ ] Test: Make a change, commit, and push

Once set up, your daily workflow is simple:
1. Edit notes in VS Code
2. `git add .`
3. `git commit -m "Description of changes"`
4. `git push`

Your notes are now backed up and accessible anywhere! 🎉
