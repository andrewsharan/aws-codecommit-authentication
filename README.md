# AWS CodeCommit Authentication Guides

This repository contains two step-by-step guides for authenticating Git clients to AWS CodeCommit. Each covers a different authentication model — pick the one that matches how your AWS account is set up.

## Contents

| File                                               | Covers                                                                                                                          |
| -------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `aws-sso-codecommit-authentication.md`             | Authenticating via **IAM Identity Center (SSO)**, using an AWS CLI profile and `git-remote-codecommit`                          |
| `aws-iam-codecommit-authentication.md` | Authenticating via **long-lived IAM user credentials** — SSH keys, HTTPS Git credentials, or HTTPS (GRC) with an IAM access key |

## File Guide

### `aws-sso-codecommit-authentication.md`

Walks through configuring `aws configure sso`, logging in with `aws sso login`, and cloning with `git-remote-codecommit` over temporary, SSO-issued credentials. Includes AWS CLI/Git/Python installation steps, the full clone-to-push workflow, and troubleshooting.

**Use this when:**
- Your AWS account uses IAM Identity Center (SSO) rather than IAM users
- You want short-lived, auto-expiring credentials instead of static ones
- You're setting up a personal developer workstation

### `aws-iam-codecommit-authentication.md`

Covers the three ways to authenticate with an **IAM user's** long-lived credentials: SSH key pairs, static HTTPS Git credentials, and HTTPS (GRC) with an IAM access key. Each method includes IAM permissions, credential generation, file permissions, testing, clone/pull/push, rotation, and troubleshooting.

**Use this when:**
- Your account doesn't have IAM Identity Center configured, or the target IAM identity is a user, not a federated one
- A tool, CI/CD system, or IDE needs a static credential (SSH key or HTTPS username/password) rather than an AWS CLI profile
- You're setting up a service account or automation that can't complete an interactive SSO browser login

## Choosing Between Them

| If you need...                                                         | Use                                                                                       |
| ---------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| The most secure, AWS-recommended setup for a human developer           | SSO guide                                                                                 |
| No AWS CLI dependency at all (e.g., a minimal CI runner)               | IAM guide → HTTPS Git credentials                                                 |
| SSH-based access (e.g., an IDE or tool that only supports SSH remotes) | IAM guide → SSH                                                                   |
| Federated/temporary credentials without an IAM user                    | SSO guide (or IAM guide's HTTPS (GRC) method with a role, if SSO isn't available) |
| Your org has already retired IAM users in favor of SSO                 | SSO guide                                                                                 |
| Your org still relies on IAM users with access keys                    | IAM guide                                                                         |


> **Note:** If you're unsure, default to the **SSO guide** — it avoids long-lived credentials and is the setup AWS currently recommends. Fall back to the **IAM guide** only when SSO isn't available or a specific tool requires a static credential.
