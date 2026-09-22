# AWS CodeCommit Authentication using IAM

This guide covers the three traditional ways to authenticate Git operations against AWS CodeCommit using long-lived IAM credentials:

1. **SSH** — an SSH key pair uploaded to an IAM user
2. **HTTPS** — a static Git username/password (an IAM service-specific credential)
3. **HTTPS (GRC)** — the `git-remote-codecommit` helper, which uses standard AWS credentials (an IAM user's access key or an assumed role) instead of a Git-specific credential


**Scope used throughout this document:**

| Setting     | Value               |
| ----------- | ------------------- |
| Region      | `ap-south-1`        |
| Repository  | `andrew-infra-repo` |
| IAM user    | `andrew`            |
| Local shell | Bash on Linux/macOS |

Replace `andrew` and `andrew-infra-repo` with your actual IAM user name and repository name where they differ.

---

## Table of Contents

1. [Method Comparison](#1-method-comparison)
2. [Common Prerequisites](#2-common-prerequisites)
3. [Method 1: SSH](#3-method-1-ssh)
4. [Method 2: HTTPS (Git Credentials)](#4-method-2-https-git-credentials)
5. [Method 3: HTTPS (GRC)](#5-method-3-https-grc)
6. [Quick Reference](#6-quick-reference)

---

## 1. Method Comparison

| Aspect                                     | SSH                     | HTTPS (Git credentials)     | HTTPS (GRC)                                             |
| ------------------------------------------ | ----------------------- | --------------------------- | ------------------------------------------------------- |
| Credential type                            | SSH key pair            | Static username/password    | AWS access key (or temporary credentials)               |
| Stored on IAM user as                      | Uploaded SSH public key | Service-specific credential | N/A — regular IAM access key                            |
| Requires AWS CLI                           | No                      | No (helper install only)    | Yes                                                     |
| Requires `git-remote-codecommit`           | No                      | No                          | Yes                                                     |
| Works with plain Git / most IDEs           | Yes                     | Yes                         | Only tools that shell out to Git with helpers installed |
| Works with federated/temporary credentials | No (IAM user only)      | No (IAM user only)          | Yes                                                     |
| Clone URL prefix                           | `ssh://`                | `https://`                  | `codecommit::`                                          |

**Clone URL reference** (region `ap-south-1`, repository `andrew-infra-repo`):

| Method      | Clone URL                                                                    |
| ----------- | ---------------------------------------------------------------------------- |
| SSH         | `ssh://git-codecommit.ap-south-1.amazonaws.com/v1/repos/andrew-infra-repo`   |
| HTTPS       | `https://git-codecommit.ap-south-1.amazonaws.com/v1/repos/andrew-infra-repo` |
| HTTPS (GRC) | `codecommit::ap-south-1://andrew-infra-repo`                                 |

---

## 2. Common Prerequisites

These apply to all three methods.

### 2.1 Repository-Level IAM Permissions

Every method still needs standard CodeCommit repository permissions on the IAM user. Attach one managed policy based on the required access level:

| Managed Policy            | Grants                                                               |
| ------------------------- | -------------------------------------------------------------------- |
| `AWSCodeCommitReadOnly`   | `GitPull` and read-only repository actions                           |
| `AWSCodeCommitPowerUser`  | `GitPull`, `GitPush`, branches, pull requests — no repository delete |
| `AWSCodeCommitFullAccess` | Full control, including repository creation and deletion             |

To scope access to a single repository instead of all repositories, use a custom policy with a `Resource` ARN such as:

```text
arn:aws:codecommit:ap-south-1:<AWS_ACCOUNT_ID>:andrew-infra-repo
```

### 2.2 Tools

```bash
git --version
aws --version   # required only for HTTPS (GRC), optional for the other two methods
```

If either is missing, install them first.

---

## 3. Method 1: SSH

### 3.1 Prerequisites and IAM Permissions

In addition to the repository permissions in [Section 2.1](#21-repository-level-iam-permissions), the IAM user needs permission to manage its own SSH public keys. Attach the AWS managed policy `IAMUserSSHKeys` (`arn:aws:iam::aws:policy/IAMUserSSHKeys`):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "iam:UploadSSHPublicKey",
        "iam:ListSSHPublicKeys",
        "iam:GetSSHPublicKey",
        "iam:UpdateSSHPublicKey",
        "iam:DeleteSSHPublicKey"
      ],
      "Resource": "arn:aws:iam::*:user/${aws:username}"
    }
  ]
}
```

Without this policy, an administrator must upload, rotate, or remove the key on the user's behalf.

> **CodeCommit SSH key constraints:** RSA keys only, minimum 2048 bits, encoded in `ssh-rsa` or PEM format. ED25519 and other key types are not accepted.

### 3.2 Credential (Key) Generation

Create the `.ssh` directory with the correct permissions:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
```

Generate the key pair:

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/codecommit_rsa
```

- `-t rsa -b 4096` — CodeCommit requires RSA, and 4096 bits exceeds the 2048-bit minimum.
- When prompted for a passphrase, press **Enter** twice for no passphrase (suitable for automation), or set one and use `ssh-agent` for interactive use.

Display the public key so it can be uploaded to IAM:

```bash
cat ~/.ssh/codecommit_rsa.pub
```

### 3.3 Configuration

**Upload the public key to IAM.**

AWS Console: **IAM → Users → andrew → Security credentials → SSH public keys for AWS CodeCommit → Upload SSH public key**, then paste the output of the `cat` command above.

**CLI equivalent:**

```bash
aws iam upload-ssh-public-key \
  --user-name andrew \
  --ssh-public-key-body "$(cat ~/.ssh/codecommit_rsa.pub)"
```

The response contains an `SSHPublicKeyId` (for example, `APKAIOSFODNN7EXAMPLE`). Record it — this ID, **not** the IAM user name, is what SSH uses for the `User` field in the next step. If you lose it, retrieve it with:

```bash
aws iam list-ssh-public-keys --user-name andrew
```

**Create `~/.ssh/config`.** Add the following block (create the file if it doesn't exist):

```bash
cat >> ~/.ssh/config << 'EOF'
Host git-codecommit.*.amazonaws.com
  User APKAIOSFODNN7EXAMPLE
  IdentityFile ~/.ssh/codecommit_rsa
EOF
```

Replace `APKAIOSFODNN7EXAMPLE` with your own SSH key ID.

### 3.4 File Permissions

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/config
chmod 600 ~/.ssh/codecommit_rsa
chmod 644 ~/.ssh/codecommit_rsa.pub
```

| Path                        | Mode  | Reason                                                                    |
| --------------------------- | ----- | ------------------------------------------------------------------------- |
| `~/.ssh`                    | `700` | OpenSSH refuses to use keys inside a group- or world-accessible directory |
| `~/.ssh/config`             | `600` | May reference private key paths; owner-only access                        |
| `~/.ssh/codecommit_rsa`     | `600` | Private key — OpenSSH rejects keys with looser permissions                |
| `~/.ssh/codecommit_rsa.pub` | `644` | Public key — safe to be world-readable                                    |

### 3.5 Authentication Testing

```bash
ssh git-codecommit.ap-south-1.amazonaws.com
```

Expected output:

```text
You have successfully authenticated over SSH. You can use Git to interact with AWS CodeCommit.
Connection to git-codecommit.ap-south-1.amazonaws.com closed by remote host.
```

> This is a **success** message, not an error. CodeCommit's SSH endpoint does not provide an interactive shell, so it authenticates and immediately closes the connection.

For detailed diagnostics:

```bash
ssh -vvv git-codecommit.ap-south-1.amazonaws.com
```

### 3.6 Repository Clone, Pull, and Push

```bash
git clone ssh://git-codecommit.ap-south-1.amazonaws.com/v1/repos/andrew-infra-repo
cd andrew-infra-repo
```

```bash
git pull
```

```bash
git add .
git commit -m "Your commit message"
git push
```

### 3.7 Credential Rotation / Revocation

Rotate before the old key is removed, so access is never interrupted:

| Step                                                                                | Command                                                                                                         |
| ----------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| 1. List current keys                                                                | `aws iam list-ssh-public-keys --user-name andrew`                                                               |
| 2. Generate a new key pair                                                          | `ssh-keygen -t rsa -b 4096 -f ~/.ssh/codecommit_rsa_new`                                                        |
| 3. Upload the new public key                                                        | `aws iam upload-ssh-public-key --user-name andrew --ssh-public-key-body "$(cat ~/.ssh/codecommit_rsa_new.pub)"` |
| 4. Update `~/.ssh/config` and `IdentityFile` to point at the new key and new key ID | —                                                                                                               |
| 5. Test authentication (Section 3.5)                                                | `ssh git-codecommit.ap-south-1.amazonaws.com`                                                                   |
| 6. Deactivate the old key (reversible)                                              | `aws iam update-ssh-public-key --user-name andrew --ssh-public-key-id <OLD_KEY_ID> --status Inactive`           |
| 7. Delete the old key once confirmed unused                                         | `aws iam delete-ssh-public-key --user-name andrew --ssh-public-key-id <OLD_KEY_ID>`                             |
| 8. Remove the old local private key file                                            | `rm ~/.ssh/codecommit_rsa`                                                                                      |

**Immediate revocation** (compromised key): skip straight to steps 6–8, or delete the key outright:

```bash
aws iam delete-ssh-public-key --user-name andrew --ssh-public-key-id <KEY_ID>
```

### 3.8 Troubleshooting and Common Errors

| Symptom                                                                    | Likely Cause                                                    | Resolution                                                                                                               |
| -------------------------------------------------------------------------- | --------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `Permission denied (publickey)`                                            | Key not uploaded, wrong `IdentityFile` path, or key deactivated | Confirm the key is `Active` (`aws iam list-ssh-public-keys`), verify the path in `~/.ssh/config`, retest with `ssh -vvv` |
| `Permissions 0644 for '.../codecommit_rsa' are too open`                   | Private key permissions too loose                               | `chmod 600 ~/.ssh/codecommit_rsa`                                                                                        |
| `fatal: Could not read from remote repository` after a successful SSH test | Repository name or region mismatch in the clone URL             | Verify the URL's region matches the repository's actual region and the repository name is exact                          |
| SSH test connects but uses the wrong key                                   | Multiple SSH keys present, agent offers the wrong one           | Add `IdentitiesOnly yes` to the `Host` block in `~/.ssh/config`                                                          |
| `Host key verification failed`                                             | Local `known_hosts` has a stale entry, or a rare true MITM      | Remove the stale entry from `known_hosts` and retry; treat repeated unexpected prompts with suspicion                    |
| Connection appears to "fail" right after the success message               | Expected behavior                                               | CodeCommit's SSH endpoint has no interactive shell; the connection closing after the success banner is normal            |
| `ssh: connect to host ... port 22: Connection timed out`                   | Outbound port 22 blocked by a firewall or corporate network     | Use HTTPS or HTTPS (GRC) instead, or request a firewall exception                                                        |

---

## 4. Method 2: HTTPS (Git Credentials)

### 4.1 Prerequisites and IAM Permissions

In addition to the repository permissions in [Section 2.1](#21-repository-level-iam-permissions), the IAM user needs permission to manage its own service-specific credentials. Attach the AWS managed policy `IAMSelfManageServiceSpecificCredentials` (`arn:aws:iam::aws:policy/IAMSelfManageServiceSpecificCredentials`):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "iam:CreateServiceSpecificCredential",
        "iam:ListServiceSpecificCredentials",
        "iam:UpdateServiceSpecificCredential",
        "iam:DeleteServiceSpecificCredential",
        "iam:ResetServiceSpecificCredential"
      ],
      "Resource": "arn:aws:iam::*:user/${aws:username}"
    }
  ]
}
```

These "HTTPS Git credentials for AWS CodeCommit" are IAM **service-specific credentials** scoped to `codecommit.amazonaws.com` — a static username/password pair usable only for CodeCommit Git operations, not for the AWS API or console.

### 4.2 Credential Generation

AWS Console: **IAM → Users → andrew → Security credentials → HTTPS Git credentials for AWS CodeCommit → Generate credentials**, then download the `.csv` file. The password is shown only once.

**CLI equivalent:**

```bash
aws iam create-service-specific-credential \
  --user-name andrew \
  --service-name codecommit.amazonaws.com
```

The response includes `ServiceUserName` and `ServicePassword` — save both immediately; the password cannot be retrieved again (only reset).

### 4.3 Configuration

No file-based configuration is required. Optionally cache the credentials so Git doesn't prompt on every operation:

```bash
git config --global credential.helper store
```

> `store` saves credentials in plain text in `~/.git-credentials`. For shared or non-encrypted disks, consider `git config --global credential.helper cache --timeout=3600` instead, which holds credentials in memory temporarily.

### 4.4 File Permissions

If using `credential.helper store`, restrict the resulting file:

```bash
chmod 600 ~/.git-credentials
```

| Path                 | Mode  | Reason                                       |
| -------------------- | ----- | -------------------------------------------- |
| `~/.git-credentials` | `600` | Contains the plaintext username and password |

### 4.5 Authentication Testing

There is no standalone connectivity test for this method — the first Git operation against the repository doubles as the test:

```bash
git ls-remote https://git-codecommit.ap-south-1.amazonaws.com/v1/repos/andrew-infra-repo
```

When prompted, enter the `ServiceUserName` and `ServicePassword` from Section 4.2. A list of refs (or no output for an empty repository) confirms success.

### 4.6 Repository Clone, Pull, and Push

```bash
git clone https://git-codecommit.ap-south-1.amazonaws.com/v1/repos/andrew-infra-repo
cd andrew-infra-repo
```

```bash
git pull
```

```bash
git add .
git commit -m "Your commit message"
git push
```

Enter the Git credential username and password at the prompt if they are not cached.

### 4.7 Credential Rotation / Revocation

An IAM user can hold a maximum of two service-specific credentials for CodeCommit at once, which allows overlap during rotation.

| Step                                                                         | Command                                                                                                                     |
| ---------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| 1. List current credentials                                                  | `aws iam list-service-specific-credentials --user-name andrew --service-name codecommit.amazonaws.com`                      |
| 2. Generate a new credential (if under the limit of two)                     | `aws iam create-service-specific-credential --user-name andrew --service-name codecommit.amazonaws.com`                     |
| 3. Update any stored credentials (`~/.git-credentials` or a secrets manager) | —                                                                                                                           |
| 4. Test with the new credential (Section 4.5)                                | —                                                                                                                           |
| 5. Deactivate the old credential (reversible)                                | `aws iam update-service-specific-credential --user-name andrew --service-specific-credential-id <OLD_ID> --status Inactive` |
| 6. Delete the old credential once confirmed unused                           | `aws iam delete-service-specific-credential --user-name andrew --service-specific-credential-id <OLD_ID>`                   |

**In-place rotation** (reuse the same username, generate a new password):

```bash
aws iam reset-service-specific-credential \
  --user-name andrew \
  --service-specific-credential-id <CREDENTIAL_ID>
```

**Immediate revocation** (compromised credential):

```bash
aws iam delete-service-specific-credential \
  --user-name andrew \
  --service-specific-credential-id <CREDENTIAL_ID>
```

### 4.8 Troubleshooting and Common Errors

| Symptom                                                         | Likely Cause                                                                                            | Resolution                                                                                                             |
| --------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `remote: Repository not found` / `fatal: Authentication failed` | Wrong username/password, or credential inactive                                                         | Re-check the `.csv` file, or reset the credential (`reset-service-specific-credential`) and retry                      |
| Git doesn't prompt, then fails silently                         | A stale cached credential in `~/.git-credentials` or the OS keychain                                    | Clear cached credentials (delete the line in `~/.git-credentials`, or clear the OS credential manager entry) and retry |
| `LimitExceededException` when generating a credential           | Two service-specific credentials already exist for this user                                            | Delete or reset an existing credential before creating a new one                                                       |
| macOS repeatedly reprompts despite `credential.helper store`    | macOS Keychain intercepting credentials from a prior `git-remote-codecommit` or credential-helper setup | Remove old entries from Keychain Access, then retry                                                                    |
| Credentials work in the browser download but not in Git         | Copy-paste introduced trailing whitespace or a line break                                               | Re-copy the username/password directly from the `.csv`, or regenerate with the CLI and copy the JSON fields exactly    |

---

## 5. Method 3: HTTPS (GRC)

### 5.1 Prerequisites and IAM Permissions

**Is HTTPS (GRC) supported?** Yes. AWS documents "HTTPS (GRC)" as one of the three official CodeCommit clone-URL types, alongside "HTTPS" and "SSH" — it refers to the `git-remote-codecommit` utility. It is AWS's own recommended method for CLI-based and federated access.

In addition to the repository permissions in [Section 2.1](#21-repository-level-iam-permissions), this method needs standard IAM access keys rather than a Git-specific credential. If self-service key rotation is required, attach a custom policy such as:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "iam:CreateAccessKey",
        "iam:ListAccessKeys",
        "iam:UpdateAccessKey",
        "iam:DeleteAccessKey",
        "iam:GetAccessKeyLastUsed"
      ],
      "Resource": "arn:aws:iam::*:user/${aws:username}"
    }
  ]
}
```

> This is a customer-managed policy, not a single AWS managed policy — unlike Sections 3.1 and 4.1, AWS does not publish one specifically for self-service access-key management.

Required tools: AWS CLI v2 and `git-remote-codecommit` (a Python package installed via `pip`).

### 5.2 Credential Generation

Generate a long-term access key for the IAM user:

AWS Console: **IAM → Users → andrew → Security credentials → Access keys → Create access key**.

CLI equivalent (requires an existing credential, such as an administrator's, to run):

```bash
aws iam create-access-key --user-name andrew
```

Record the `AccessKeyId` and `SecretAccessKey` immediately — the secret is shown only once.

Install `git-remote-codecommit`:

```bash
pip install git-remote-codecommit --break-system-packages
```

> If your system restricts direct `pip` installs, use a virtual environment instead — see your organization's Python packaging guide.

### 5.3 Configuration

Configure an AWS CLI profile with the access key:

```bash
aws configure --profile andrew-codecommit
```

Enter the `AccessKeyId`, `SecretAccessKey`, default region (`ap-south-1`), and output format (`json`) at the prompts.

Verify the profile:

```bash
aws sts get-caller-identity --profile andrew-codecommit
```

### 5.4 File Permissions

`aws configure` writes credentials to `~/.aws/credentials` and `~/.aws/config`. AWS CLI sets restrictive permissions on these automatically, but confirm them:

```bash
chmod 600 ~/.aws/credentials
chmod 600 ~/.aws/config
```

| Path                 | Mode  | Reason                                                     |
| -------------------- | ----- | ---------------------------------------------------------- |
| `~/.aws/credentials` | `600` | Contains the plaintext access key ID and secret access key |
| `~/.aws/config`      | `600` | Contains profile and region configuration                  |

### 5.5 Authentication Testing

```bash
aws sts get-caller-identity --profile andrew-codecommit
```

```bash
aws codecommit get-repository \
  --repository-name andrew-infra-repo \
  --profile andrew-codecommit \
  --region ap-south-1
```

A successful response returns repository metadata, confirming both the AWS credentials and CodeCommit authorization.

### 5.6 Repository Clone, Pull, and Push

```bash
git clone codecommit::ap-south-1://andrew-codecommit@andrew-infra-repo
cd andrew-infra-repo
```

```bash
git pull
```

```bash
git add .
git commit -m "Your commit message"
git push
```

> The `andrew-codecommit@` segment names the AWS CLI profile to use. Omit it (`codecommit::ap-south-1://andrew-infra-repo`) to use the `default` profile instead.

### 5.7 Credential Rotation / Revocation

| Step                                          | Command                                                                                       |
| --------------------------------------------- | --------------------------------------------------------------------------------------------- |
| 1. List current access keys                   | `aws iam list-access-keys --user-name andrew`                                                 |
| 2. Create a new access key (max two per user) | `aws iam create-access-key --user-name andrew`                                                |
| 3. Update the CLI profile with the new key    | `aws configure --profile andrew-codecommit`                                                   |
| 4. Test with the new key (Section 5.5)        | —                                                                                             |
| 5. Deactivate the old key (reversible)        | `aws iam update-access-key --user-name andrew --access-key-id <OLD_KEY_ID> --status Inactive` |
| 6. Delete the old key once confirmed unused   | `aws iam delete-access-key --user-name andrew --access-key-id <OLD_KEY_ID>`                   |

**Immediate revocation** (compromised key):

```bash
aws iam update-access-key --user-name andrew --access-key-id <KEY_ID> --status Inactive
aws iam delete-access-key --user-name andrew --access-key-id <KEY_ID>
```

### 5.8 Troubleshooting and Common Errors

| Symptom                                                     | Likely Cause                                                        | Resolution                                                                                     |
| ----------------------------------------------------------- | ------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `git: 'remote-codecommit' is not a git command`             | `git-remote-codecommit` not installed or not on `PATH`              | Reinstall with `pip install git-remote-codecommit`, confirm with `which git-remote-codecommit` |
| `fatal: repository 'codecommit::...' does not exist`        | Wrong region, repository name, or profile in the URL                | Re-check each segment of the `codecommit::<region>://<profile>@<repo>` URL                     |
| `Unable to locate credentials`                              | Named profile missing or misspelled                                 | Run `aws configure list-profiles` and confirm the profile name matches the URL                 |
| `AccessDeniedException` on `git push`                       | IAM user has `GitPull` but not `GitPush`                            | Attach `AWSCodeCommitPowerUser` or an equivalent custom policy                                 |
| Works with `aws sts get-caller-identity` but not with `git` | Access key is valid but lacks CodeCommit permissions                | Confirm the repository-level IAM policy from Section 2.1 is attached                           |
| `InvalidAccessKeyId` / `SignatureDoesNotMatch`              | Access key deactivated, deleted, or mistyped during `aws configure` | Verify key status with `aws iam list-access-keys`, re-run `aws configure`                      |

---

## 6. Quick Reference

| Task                | SSH                                                                                  | HTTPS (Git credentials)                                                                                 | HTTPS (GRC)                                                              |
| ------------------- | ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| Generate credential | `ssh-keygen -t rsa -b 4096 -f ~/.ssh/codecommit_rsa`                                 | `aws iam create-service-specific-credential --user-name andrew --service-name codecommit.amazonaws.com` | `aws iam create-access-key --user-name andrew`                           |
| Test                | `ssh git-codecommit.ap-south-1.amazonaws.com`                                        | `git ls-remote https://git-codecommit.ap-south-1.amazonaws.com/v1/repos/andrew-infra-repo`              | `aws sts get-caller-identity --profile andrew-codecommit`                |
| Clone               | `git clone ssh://git-codecommit.ap-south-1.amazonaws.com/v1/repos/andrew-infra-repo` | `git clone https://git-codecommit.ap-south-1.amazonaws.com/v1/repos/andrew-infra-repo`                  | `git clone codecommit::ap-south-1://andrew-codecommit@andrew-infra-repo` |
| Revoke              | `aws iam delete-ssh-public-key --user-name andrew --ssh-public-key-id <ID>`          | `aws iam delete-service-specific-credential --user-name andrew --service-specific-credential-id <ID>`   | `aws iam delete-access-key --user-name andrew --access-key-id <ID>`      |
