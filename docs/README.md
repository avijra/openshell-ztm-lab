# OpenShell on Zero Trust — Demo Lab

> **Securing AI Agents in Production with Zero Trust on ROSA**

Welcome to the hands-on demo lab for **NVIDIA OpenShell** running on Red Hat's **Zero Trust Validated Pattern (ZTVP)** on **ROSA** (Red Hat OpenShift Service on AWS).

## The Problem: AI Agents Are Powerful — and Dangerous

AI coding agents like [Pi](https://github.com/NVIDIA/OpenShell) are revolutionizing software development. They can write code, install packages, call APIs, and run programs — all autonomously. But this power creates serious security risks:

| What an unconstrained agent can do | Real-world risk |
|-------------------------------------|-----------------|
| Call any network endpoint | **Data exfiltration** — agent sends your source code to an attacker's server |
| Read any file on disk | **Secret theft** — agent reads `/etc/shadow`, K8s service account tokens, or `.env` files |
| Access raw API keys | **Credential leakage** — agent logs or transmits your `sk-...` key |
| Install arbitrary packages | **Supply chain attack** — agent installs a backdoored npm package |
| Write to system paths | **Persistence** — agent modifies binaries or creates cron jobs |

**Zero Trust for AI Agents** solves this. Instead of trusting the agent and hoping for the best, we **enforce explicit policy on every action**. The agent can only reach approved endpoints, read approved files, and use credentials it never actually sees.

## What Is Zero Trust?

**Zero Trust** is a security model where **nothing is trusted by default** — not the network, not the user, not the process. Every request must be explicitly authorized.

Traditional security puts a firewall around the perimeter and trusts everything inside. Zero Trust assumes the attacker is *already inside* and locks down every interaction individually.

In this lab, Zero Trust means:
- **Network**: The agent can only reach 3 specific hosts. Everything else is blocked at the kernel level.
- **Filesystem**: The agent can only read/write to 2 directories. Everything else returns `Permission denied`.
- **Credentials**: The agent never sees real API keys — only opaque tokens that are useless outside the cluster.

## What You Will Learn

In this lab you will:

1. **Deploy** an AI agent sandbox from the OpenShift Developer Catalog — one click
2. **Run** the Pi coding agent powered by DeepSeek LLM inside a zero-trust sandbox
3. **Prove** that network access is restricted to only approved endpoints
4. **Prove** that filesystem access is locked down by Landlock LSM (a Linux kernel security module)
5. **Observe** that API credentials are never visible inside the sandbox
6. **Explore** how Vault, SPIFFE, ACS, and External Secrets integrate as defense-in-depth
7. **Update** the security policy live without redeploying anything

## The Key Technologies

| Technology | What it is | Role in this lab |
|------------|-----------|-----------------|
| **ROSA** | Red Hat OpenShift Service on AWS — a managed Kubernetes platform | The cluster running everything |
| **ZTVP** | Zero Trust Validated Pattern — a curated set of security tools deployed together | Provides Vault, ESO, Keycloak, SPIFFE, and ACS |
| **OpenShell** | NVIDIA's framework for running AI agents in secure sandboxes | Enforces network, filesystem, and credential policies |
| **Pi** | NVIDIA's agentic coding assistant (Rust binary) | The AI agent writing and running code inside the sandbox |
| **DeepSeek** | A large language model (LLM) API | The "brain" that generates code when Pi asks for it |
| **Landlock LSM** | A Linux kernel security module for filesystem restrictions | Enforces which files the agent can read/write — at the kernel level |
| **Helm** | A package manager for Kubernetes | How we deploy OpenShell in one click from the OpenShift catalog |

## Lab Duration

| Section | Time | What you'll do |
|---------|------|---------------|
| Architecture | 5 min | Understand how the pieces fit together |
| Prerequisites | 3 min | Verify your environment |
| Deploy from Catalog | 5 min | Install OpenShell from the OpenShift Console |
| Client Setup | 2 min | Configure the CLI |
| AI Agent Coding | 5 min | Watch Pi write real code with DeepSeek |
| Zero Trust Demos | 10 min | Prove network + filesystem + credential restrictions |
| ZTVP Integration | 5 min | Explore Vault, SPIFFE, ACS, Keycloak, ESO |
| Dynamic Policy | 3 min | Update the policy live |
| **Total** | **~40 min** | |

## Fill In Your Environment

Use the **configuration bar** at the top of the page to enter your environment details. All commands in this lab will **auto-populate** with your values — no copy-paste errors.

> **Tip:** Your instructor will provide the Bastion Host, SSH Password, Cluster Domain, and DeepSeek API Key. Fill them in before starting Module 2.

## Ready?

Let's start by understanding the architecture.

Proceed to: **Architecture** →
