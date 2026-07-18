## Step 1: Create the Gitea Personal Access Token (PAT)
_This token allows GitHub Actions to securely push code into your Gitea account._
1. Log into your account on [gitea.com](https://gitea.com)
2. Click your profile picture (top-right) > Settings.
3. Click the **Applications** tab in the left sidebar.
4. Under **Manage Access Tokens**, type a token name (e.g., `PAT-migrate`)
5. Ensure the token has **Read and Write** permissions for **Repositories**
6. Click Generate Token.
7. **Copy this token immediately** and note it down. (It will disappear forever once you leave the page).

## Step 2: Set Up Your Project Locally & Push to GitHub
1. Create a brand new repository on **GitHub**. Leave it empty (do not add a `README` or `.gitignore` yet).
2. Open your terminal inside your project folder on your local machine and run:
```bash
git init
git checkout -b development
git add .
git commit -m "Initial commit"
git remote add origin https://github.com
git push -u origin development

```

## Step 3: Migrate the Repository from GitHub to Gitea
1. Log into [gitea.com](https://gitea.com)
2. Click the `+` icon in the top-right corner of the navbar > select **New Migration**
3. Select **GitHub**.
4. Fill out the form:
- **Migrate / From URL**: `https://github.com/username/repo-name.git`
- **Username**: Your GitHub username.
- **Password**: Your GitHub Personal Access Token.
5. Click **Migrate Repository**.

## Step 4: Configure Local Git for "Dual-Push" Prompt Bypassing
_This stops VS Code and Linux from asking for your password every time you push._
1. Force Git to permanently remember your credentials in a hidden file:
```bash
git config --global credential.helper store
```
2. Set up your local **origin** remote to broadcast changes to both GitHub and Gitea simultaneously:
```bash
git remote set-url origin https://github.com/username/repo-name.git

git remote set-url --add --push origin https://github.com/username/repo-name.git

git remote set-url --add --push origin https://gitea.com/username/repo-name.git
```
3. Verify local remote endpoints:
```bash
git remote -v
```
You should see:
```bash
origin  https://github.com/username/repo-name.git (fetch)
origin  https://github.com/username/repo-name.git (push)
origin  https://gitea.com/username/repo-name.git (push)
```
4. Trigger a manual push to save your credentials into the store one last time:
```bash
git push
```
- Enter your Gitea **username** and paste the Gitea token you created in Step 1 when prompted.

## Step 5: Save Your Token in GitHub Repository Secrets
1. Go to your repository on **GitHub.com** > click **Settings** in the top tab bar.
2. In the left sidebar, click **Secrets and variables** > select **Actions**.
3. Click the **New repository secret** button.
4. Set the name to exactly: `GITEA_TOKEN`
5. Paste your **Gitea Personal Access Token** (from Step 1) into the value box.
6. Click **Add secret**.

## Step 6: Create the GitHub Actions Auto-Sync Workflow File
1. In your local project using VS Code, create a folder structure and file at this exact path: `.github/workflows/sync-to-gitea.yaml`
2. Paste this exact configuration inside that file:
```bash
name: Sync to Gitea on Merge

on:
  push:
    branches:
      - main
      - development  # Add any other core tracking branches here

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Crucial to fetch all history and branches

      - name: Push to Gitea
        env:
          GITEA_PAT: ${{ secrets.GITEA_TOKEN }}
        run: |
          B64_AUTH=$(echo -n "username:${GITEA_PAT}" | base64 | tr -d '\n')
          git remote add gitea https://gitea.com/username/repo-name.git
          git config --local http.extraheader "Authorization: Basic ${B64_AUTH}"
          git push --force gitea --all
          git push --force gitea --tags

```
3. Commit and push this workflow to GitHub:
```bash
git add .github/workflows/sync-to-gitea.yml
git commit -m "ci: add automated gitea mirror pipeline"
git push
```

## Step 7: Disable the Actions Tab inside Gitea
_This clears out the "No matching online runner" warning message on your Gitea dashboard._
1. Open your repository on **Gitea.com**.
2. Click **Settings** in the top navbar.
3. Click the **Actions > General** tab on the left sidebar menu.
4. Check **Disable Repository Actions**.
5. Click **Update Settings**.

## Step 8: Manual Branch Creation & Sync Workflow
_Run these commands sequentially whenever you start working on a new feature branch._
1. Create a brand-new branch in your **GitHub UI** (keeping the `development` branch as the source).
2. Open your local terminal or VS Code and execute the following exact commands in sequence:
```bash
git switch development
git fetch --all
git pull
git switch name-of-your-new-feature-branch
git push -u origin <name-of-your-new-feature-branch
```

_(GitHub will safely report "Everything up-to-date", while Gitea will receive the branch cleanly for the very first time. From this point on, running `git push` on this branch will update both platforms concurrently)._
