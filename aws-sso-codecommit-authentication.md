# AWS CodeCommit Authentication Using IAM Identity Center

This guide explains how to authenticate to AWS CodeCommit using **AWS IAM Identity Center (SSO)**, an **AWS CLI profile**, and **`git-remote-codecommit`**. The setup relies solely on temporary credentials and does not require IAM access keys, CodeCommit Git credentials, or SSH keys.

## Table of Contents

1. [Overview](#1-overview)
2. [Prerequisites](#2-prerequisites)
3. [Configure the AWS CLI for IAM Identity Center](#3-configure-the-aws-cli-for-iam-identity-center)
4. [Authenticate and Verify AWS Access](#4-authenticate-and-verify-aws-access)
5. [Install `git-remote-codecommit`](#5-install-git-remote-codecommit)
6. [Clone and Initialize the Repository](#6-clone-and-initialize-the-repository)
7. [Verify the Complete Setup](#7-verify-the-complete-setup)
8. [Daily Workflow](#8-daily-workflow)
9. [Final Architecture](#9-final-architecture)
10. [Troubleshooting](#10-troubleshooting)
11. [Quick Reference](#11-quick-reference)

---

## 1. Overview

### 1.1 Authentication Flow

Git access to CodeCommit follows this chain:

```text
Developer
   │
   │  aws sso login
   ▼
IAM Identity Center
   │
   │  Temporary credentials
   ▼
AWS CLI profile
   │
   │  git-remote-codecommit
   ▼
AWS CodeCommit
   │
   ▼
Repository
```

### 1.2 Environment Reference

The following values are used throughout this guide.

| Setting                    | Value                                           |
| -------------------------- | ----------------------------------------------- |
| IAM Identity Center region | `ap-south-1`                                    |
| SSO session name           | `andrew`                                        |
| SSO start URL              | `https://<SSO_DIRECTORY_ID>.awsapps.com/start/` |
| AWS account                | `<AWS_ACCOUNT_ID>`                              |
| Permission set             | `AdministratorAccess`                           |
| AWS CLI profile            | `andrew-admin`                                  |
| CLI default region         | `ap-south-1`                                    |
| CLI output format          | `json`                                          |
| CodeCommit region          | `ap-south-1`                                    |
| Remote Repository                 | `andrew-infra-repo`                      |
| Local repository path      | `~/home/andrew/andrew-infra-repo`                           |

> **Note:** Sensitive values are shown as placeholders. Replace `<AWS_ACCOUNT_ID>` with your 12-digit AWS account ID and `<SSO_DIRECTORY_ID>` with the directory identifier from your AWS access portal URL.

---

## 2. Prerequisites

This section installs the three tools required for the rest of this guide: **AWS CLI v2**, **Git**, and **Python 3**. The commands target **Ubuntu** and assume:

- A terminal session on an Ubuntu machine
- A user account with `sudo` privileges
- An active internet connection

### 2.1 Check What Is Already Installed

Run each check command below. If a tool already meets the requirement, skip its installation step. If a command returns `command not found`, or `aws --version` does not report `aws-cli/2.x.x`, install the tool using the linked step.

| Tool    | Check Command       | Requirement                  | Installation Step                     |
| ------- | ------------------- | ---------------------------- | ------------------------------------- |
| AWS CLI | `aws --version`     | AWS CLI v2 (`aws-cli/2.x.x`) | [Section 2.3](#2.3-install-aws-cli-v2) |
| Git     | `git --version`     | Git installed                | [Section 2.4](#2.4-install-git)        |
| Python  | `python3 --version` | Python 3                     | [Section 2.5](#2.5-install-python-3)   |

### 2.2 Update the Package Index

Refresh Ubuntu's package list before installing anything:

```bash
sudo apt update
```


### 2.3 Install AWS CLI v2

Install the AWS CLI with AWS's official installer. Complete the steps in order.

#### Step 1: Install the Download Utilities

```bash
sudo apt install -y curl unzip
```

#### Step 2: Identify Your CPU Architecture

```bash
uname -m
```

Use the result to choose the installer file in the next step:

| Output    | Architecture         | Installer File                 |
| --------- | -------------------- | ------------------------------ |
| `x86_64`  | Intel / AMD (64-bit) | `awscli-exe-linux-x86_64.zip`  |
| `aarch64` | ARM (64-bit)         | `awscli-exe-linux-aarch64.zip` |

#### Step 3: Download the Installer

Work in a temporary directory so the downloaded files are easy to remove afterward:

```bash
cd /tmp
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
```

> **ARM systems:** In the URL above, replace `awscli-exe-linux-x86_64.zip` with `awscli-exe-linux-aarch64.zip`.

#### Step 4: Extract and Run the Installer

```bash
unzip -q awscliv2.zip
sudo ./aws/install
```

#### Step 5: Verify the Installation

```bash
aws --version
```

The output should begin with `aws-cli/2.`:

```text
aws-cli/2.x.x ...
```

#### Step 6: Clean Up

```bash
rm -rf awscliv2.zip aws
cd ~
```

> **Note:** To update the AWS CLI later, repeat Steps 3 and 4 and add the `--update` flag to the installer command: `sudo ./aws/install --update`.

### 2.4 Install Git

```bash
sudo apt install -y git
git --version
```

The output should be similar to:

```text
git version 2.x.x
```

Git records a name and email address with every commit, so configure them before your first commit ([Section 6.5](#65-stage-and-commit-the-files)). Replace the sample values with your own:

```bash
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
```

Verify the configuration:

```bash
git config --global --list
```

### 2.5 Install Python 3

Ubuntu typically includes Python 3 already, so this command may report that it is installed. It also installs `python3-venv`, which is required to create the virtual environment in [Section 5](#5-install-git-remote-codecommit):

```bash
sudo apt install -y python3 python3-venv
python3 --version
```

The output should be similar to:

```text
Python 3.x.x
```

### 2.6 Verify All Installations

Run all three checks:

```bash
aws --version
git --version
python3 --version
```

| Tool    | Expected Output     |
| ------- | ------------------- |
| AWS CLI | `aws-cli/2.x.x ...` |
| Git     | `git version 2.x.x` |
| Python  | `Python 3.x.x`      |

Once all three commands return a version, continue to [Section 3](#3-configure-the-aws-cli-for-iam-identity-center).

---

## 3. Configure the AWS CLI for IAM Identity Center

### 3.1 Start the Configuration Wizard

```bash
aws configure sso
```

### 3.2 Enter the Wizard Values

Respond to the prompts in the following order.

| Step | Prompt                                                 | Value                                           | Notes                                                                                     |
| ---- | ------------------------------------------------------ | ----------------------------------------------- | ----------------------------------------------------------------------------------------- |
| 1    | `SSO session name`                                     | `andrew`                                        |  SSO session name                                                                                         |
| 2    | `SSO start URL`                                        | `https://<SSO_DIRECTORY_ID>.awsapps.com/start/` | Your organization's AWS access portal |
| 3    | `SSO region`                                           | `ap-south-1`                                    | The region where IAM Identity Center is configured. **Do not use `ap-south-1` here.**   |
| 4    | `SSO registration scopes`                              | `sso:account:access`                            | Accept the default                                                                     |
| 5    | Browser sign-in                                        | n/a                                             | The CLI opens your browser. Complete the IAM Identity Center authentication.              |
| 6    | AWS account selection                                  | `<AWS_ACCOUNT_ID>`                              | Select the account from the list displayed after authentication                         |
| 7    | Permission set selection                               | `AdministratorAccess`                           | Select the permission set assigned to your user                                 |
| 8    | `Default client Region [None]:`                        | `ap-south-1`                                  | The region of the CodeCommit repository                                     |
| 9    | `CLI default output format [None]:`                    | `json`                                          |          CLI Output format                                                                                 |
| 10   | `Profile name [AdministratorAccess-<AWS_ACCOUNT_ID>]:` | `andrew-admin`                                  | Replaces the suggested default name                                     |

> **Best practice:** In production environments, use a dedicated least-privilege permission set for CodeCommit instead of `AdministratorAccess`.

---

## 4. Authenticate and Verify AWS Access

### 4.1 Confirm the Profile Exists

```bash
aws configure list-profiles
```

The output should include:

```text
andrew-admin
```

### 4.2 Sign In to IAM Identity Center

```bash
aws sso login --profile andrew-admin
```

A browser window opens. Complete your organization's IAM Identity Center sign-in. On success, the AWS CLI caches the temporary SSO credentials locally.

### 4.3 Verify the AWS Identity

```bash
aws sts get-caller-identity --profile andrew-admin
```

The output should be similar to:

```json
{
    "UserId": "AROAXXXXXXXXXXXXXXXX:your-user",
    "Account": "<AWS_ACCOUNT_ID>",
    "Arn": "arn:aws:sts::<AWS_ACCOUNT_ID>:assumed-role/AWSReservedSSO_AdministratorAccess_XXXXXXXX/your-user"
}
```

The key value is `"Account": "<AWS_ACCOUNT_ID>"`. A matching account confirms that IAM Identity Center, the AWS CLI SSO configuration, and the `andrew-admin` profile are issuing temporary credentials correctly.

### 4.4 Verify CodeCommit Access

> **Tip:** Run this check before configuring Git. It is a useful troubleshooting checkpoint.

```bash
aws codecommit get-repository \
  --repository-name andrew-infra-repo \
  --profile andrew-admin \
  --region ap-south-1
```

The command should return repository metadata, for example:

```json
{
    "repositoryMetadata": {
        "accountId": "<AWS_ACCOUNT_ID>",
        "repositoryName": "andrew-infra-repo",
        "cloneUrlHttp": "https://git-codecommit.ap-south-1.amazonaws.com/v1/repos/andrew-infra-repo",
        "cloneUrlSsh": "ssh://git-codecommit.ap-south-1.amazonaws.com/v1/repos/andrew-infra-repo"
    }
}
```

If this command succeeds, AWS authentication and CodeCommit authorization are working.

### 4.5 Check Whether the SSO Session Is Still Valid

If you are unsure whether your SSO session is active, run:

```bash
aws sts get-caller-identity --profile andrew-admin
```

- If the command succeeds, your SSO credentials are available.
- If it returns an authentication or expiration error, sign in again and retry:

```bash
aws sso login --profile andrew-admin
aws sts get-caller-identity --profile andrew-admin
```

---

## 5. Install `git-remote-codecommit`

### 5.1 Why It Is Required

IAM Identity Center provides **temporary AWS credentials**. `git-remote-codecommit` lets Git use those credentials through the AWS CLI profile:

```text
IAM Identity Center → Temporary credentials → AWS CLI profile → git-remote-codecommit → CodeCommit
```

The following are **not** required for this setup, so do not create them:

- IAM access keys and secret access keys
- CodeCommit Git username and password
- SSH keys
- `~/.git-credentials`
- Hard-coded AWS credentials

### 5.2 Check Python Virtual Environment Support

Modern Ubuntu releases may prevent installing Python packages directly into the system Python environment, so use a virtual environment. Check that `venv` is available:

```bash
python3 -m venv --help
```

- If the help output appears, continue to the next step.
- If an error states that `ensurepip` is unavailable, install the Ubuntu `venv` package (for example, on Ubuntu with Python 3.12), then retry:

```bash
sudo apt install -y python3.12-venv
python3 -m venv --help
```

### 5.3 Create the Virtual Environment

```bash
python3 -m venv ~/.venvs/codecommit
```

This creates the directory `~/.venvs/codecommit/`.

### 5.4 Activate the Virtual Environment

```bash
source ~/.venvs/codecommit/bin/activate
```

The shell prompt changes from:

```text
~$
```

to:

```text
(codecommit) ~$
```

The `(codecommit)` prefix confirms that the virtual environment is active.

### 5.5 Install the Package

With the environment active, run:

```bash
python -m pip install git-remote-codecommit
```

The output should end with a message similar to:

```text
Successfully installed ... git-remote-codecommit-...
```

### 5.6 Verify the Installation

```bash
which git-remote-codecommit
```

Expected output:

```text
/home/test/.venvs/codecommit/bin/git-remote-codecommit
```

Then run:

```bash
git-remote-codecommit --help
```

The following message is expected:

```text
Too few arguments. This hook requires the git command and remote.
```

This is normal. It confirms that the executable was found and started. Git normally supplies its arguments.

### 5.7 Make the Command Permanently Available

To avoid activating the virtual environment each time you use CodeCommit, add its `bin` directory to your `PATH`:

```bash
echo 'export PATH="$HOME/.venvs/codecommit/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

Verify the command is found:

```bash
which git-remote-codecommit
```

The output should still be:

```text
/home/test/.venvs/codecommit/bin/git-remote-codecommit
```

Deactivate the virtual environment and verify again:

```bash
deactivate
which git-remote-codecommit
```

The command should still be found.

> **Note:** Prepending the directory to `PATH` also places the virtual environment's `python` and `pip` ahead of the system versions in new shells. To avoid this, append the directory instead:
>
> ```bash
> echo 'export PATH="$PATH:$HOME/.venvs/codecommit/bin"' >> ~/.bashrc
> ```

### 5.8 Virtual Environment Commands

Do **not** delete or recreate the environment each time. It remains at `~/.venvs/codecommit`.

| Action                              | Command                                   |
| ----------------------------------- | ----------------------------------------- |
| Enter (or re-enter) the environment | `source ~/.venvs/codecommit/bin/activate` |
| Exit the environment                | `deactivate`                              |

### 5.9 Virtual Environment vs. AWS SSO Session

The Python virtual environment and the AWS SSO session are **two independent things**.

| Aspect    | Python Virtual Environment                               | AWS SSO Session                        |
| --------- | -------------------------------------------------------- | -------------------------------------- |
| Purpose   | Provides Python packages such as `git-remote-codecommit` | Authenticates you to AWS               |
| Command   | `source ~/.venvs/codecommit/bin/activate`                | `aws sso login --profile andrew-admin` |
| Indicator | `(codecommit)` shell prompt prefix                       | `aws sts get-caller-identity` succeeds |

As a result, the `(codecommit)` environment can be active while you are not logged in to AWS, and you can be logged in to AWS without the environment being active.

---

## 6. Clone and Initialize the Repository

### 6.1 Determine the Git Remote URL

CodeCommit remote URLs use the following format:

```text
codecommit::<region>://<profile>@<repository>
```

| Component       | Value               |
| --------------- | ------------------- |
| Region          | `ap-south-1`        |
| AWS CLI profile | `andrew-admin`      |
| Repository      | `andrew-infra-repo` |

The resulting URL is:

```text
codecommit::ap-south-1://andrew-admin@andrew-infra-repo
```

### 6.2 Clone the Repository

```bash
git clone codecommit::ap-south-1://andrew-admin@andrew-infra-repo
```

If the repository is new and empty, the expected output is:

```text
Cloning into 'andrew-infra-repo'...
warning: You appear to have cloned an empty repository.
```

This is **normal**. It confirms that authentication succeeded and CodeCommit allowed Git to access the repository.

### 6.3 Configure the Local Repository

Enter the repository and check its status:

```bash
cd andrew-infra-repo
git status
```

On a new repository, the initial output is similar to:

```text
On branch master

No commits yet

nothing to commit
```

Rename the default branch to `main` and verify:

```bash
git branch -M main
git status
```

The output should include:

```text
On branch main
```

Verify the CodeCommit remote:

```bash
git remote -v
```

Expected output:

```text
origin  codecommit::ap-south-1://andrew-admin@andrew-infra-repo (fetch)
origin  codecommit::ap-south-1://andrew-admin@andrew-infra-repo (push)
```

This confirms that Git uses `git-remote-codecommit`, which authenticates with the `andrew-admin` AWS CLI profile to reach CodeCommit.

### 6.4 Create the Initial Repository Structure

For this infrastructure repository, create the following layout:

```bash
mkdir -p templates parameters
touch README.md
```

```text
andrew-infra-repo/
├── README.md
├── templates/
└── parameters/
```

> **Important:** Git does not track empty directories. To make the directories exist in Git immediately, add placeholder `.gitkeep` files:
>
> ```bash
> touch templates/.gitkeep parameters/.gitkeep
> ```

### 6.5 Stage and Commit the Files

Check the working tree:

```bash
git status
```

The output should be similar to the following (the directories appear when `.gitkeep` files exist):

```text
On branch main

Untracked files:
  README.md
  parameters/
  templates/
```

Stage the files:

```bash
git add .
git status
```

The files should now appear under `Changes to be committed`.

Create the initial commit with a descriptive message:

```bash
git commit -m "Initialize infra repo"
```

The output should be similar to:

```text
[main (root-commit) XXXXXXX] Initialize infra repo
```

### 6.6 Push the `main` Branch

```bash
git push -u origin main
```

The output should include:

```text
[new branch] main -> main
branch 'main' set up to track 'origin/main'
```

This creates the remote `main` branch in CodeCommit.

### 6.7 Verify the Branch and Repository Status

```bash
git branch -a
```

Expected output:

```text
* main
  remotes/origin/main
```

```bash
git status
```

Expected output:

```text
On branch main
Your branch is up to date with 'origin/main'.

nothing to commit, working tree clean
```

Your local Git repository is now fully connected to CodeCommit.

---

## 7. Verify the Complete Setup

Run the following commands:

```bash
aws sts get-caller-identity --profile andrew-admin
which git-remote-codecommit
cd ~/andrew-infra-repo
git remote -v
git branch -a
git status
```

Confirm the results against the expected values:

| Check         | Expected Result                                             |
| ------------- | ----------------------------------------------------------- |
| AWS account   | `<AWS_ACCOUNT_ID>`                                          |
| Git helper    | `/home/test/.venvs/codecommit/bin/git-remote-codecommit`    |
| Remote        | `codecommit::ap-south-1://andrew-admin@andrew-infra-repo` |
| Branch        | `main`                                                      |
| Remote branch | `origin/main`                                               |
| Working tree  | Clean                                                       |

> **Result:** The setup is validated end to end: SSO login → temporary credentials → CodeCommit API access → Git clone → initial commit → push to `main`.

---

## 8. Daily Workflow

Do **not** clone the repository again. After the initial setup, use the following workflow.

### 8.1 Standard Workflow

| Step | Action                                  | Command                                           |
| ---- | --------------------------------------- | ------------------------------------------------- |
| 1    | Open the repository                     | `cd ~/andrew-infra-repo`                          |
| 2    | Renew the SSO session (only if expired) | `aws sso login --profile andrew-admin`            |
| 3    | Pull the latest changes                 | `git pull`                                        |
| 4    | Make your changes                       | n/a                                               |
| 5    | Review the changes                      | `git status`                                      |
| 6    | Stage the changes                       | `git add .`                                       |
| 7    | Commit the changes                      | `git commit -m "Update CloudFormation templates"` |
| 8    | Push to CodeCommit                      | `git push`                                        |

### 8.2 Starting in a New Terminal

Because the virtual environment's `bin` directory is on your `PATH` (see [Section 5.7](#57-make-the-command-permanently-available)), you do not need to activate the environment. In a new terminal:

```bash
cd ~/andrew-infra-repo
aws sts get-caller-identity --profile andrew-admin
```

If the SSO session has expired, sign in again:

```bash
aws sso login --profile andrew-admin
```

Then pull the latest changes and work as usual:

```bash
git pull
```

> **Note:** If you did **not** add the virtual environment to your `PATH`, activate it before using CodeCommit:
>
> ```bash
> source ~/.venvs/codecommit/bin/activate
> ```

---

## 9. Final Architecture

The completed environment looks like this:

```text
┌─────────────────────────────┐
│ IAM Identity Center         │
│ ap-south-1                  │
└──────────────┬──────────────┘
               │ SSO login
               ▼
┌─────────────────────────────┐
│ AWS CLI profile             │
│ andrew-admin                │
└──────────────┬──────────────┘
               │ Temporary credentials
               ▼
┌─────────────────────────────┐
│ git-remote-codecommit       │
└──────────────┬──────────────┘
               │ ap-south-1
               ▼
┌─────────────────────────────┐
│ AWS CodeCommit              │
│ andrew-infra-repo           │
└──────────────┬──────────────┘
               │ origin/main
               ▼
┌─────────────────────────────┐
│ Local Git repository        │
│ ~/andrew-infra-repo         │
└─────────────────────────────┘
```

---

## 10. Troubleshooting

| Symptom                                                        | Likely Cause                                                                          | Resolution                                                                                                                                  |
| -------------------------------------------------------------- | ------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| `ensurepip` is unavailable when creating a virtual environment | The Ubuntu `venv` package is missing                                                  | Run `sudo apt install -y python3.12-venv`, then retry                                                                                       |
| `sudo apt update` fails                                        | An unrelated broken APT repository                                                    | Do not modify it for CodeCommit, resolve it separately                                                                                      |
| `git-remote-codecommit --help` prints `Too few arguments...`   | Expected behavior: the executable was found and started                               | No action required                                                                                                                          |
| `fatal: Unable to find remote helper for 'codecommit'`         | `git-remote-codecommit` is neither on the `PATH` nor in an active virtual environment | Activate the virtual environment or add it to the `PATH` (see [Section 5.7](#57-make-the-command-permanently-available))                    |
| Authentication or expiration errors from the AWS CLI           | The SSO session has expired                                                           | Run `aws sso login --profile andrew-admin`                                                                                                  |
| `aws codecommit get-repository` fails                          | Inactive SSO session, wrong region, or insufficient permissions                       | Verify the session with `aws sts get-caller-identity`, use `--region ap-south-1`, and confirm the permission set allows CodeCommit access |
| `warning: You appear to have cloned an empty repository.`      | The repository is new and empty                                                       | Expected; continue with [Section 6.3](#63-configure-the-local-repository)                                                                   |

---

## 11. Quick Reference

| Task                                        | Command                                              |
| ------------------------------------------- | ---------------------------------------------------- |
| Log in to AWS                               | `aws sso login --profile andrew-admin`               |
| Verify AWS identity                         | `aws sts get-caller-identity --profile andrew-admin` |
| Enter the repository                        | `cd ~/andrew-infra-repo`                             |
| Pull changes                                | `git pull`                                           |
| Check changes                               | `git status`                                         |
| Stage changes                               | `git add .`                                          |
| Commit changes                              | `git commit -m "Your commit message"`                |
| Push changes                                | `git push`                                           |
| Exit the Python environment (if activated)  | `deactivate`                                         |
| Re-enter the Python environment (if needed) | `source ~/.venvs/codecommit/bin/activate`            |
| Verify the CodeCommit Git helper            | `which git-remote-codecommit`                        |
