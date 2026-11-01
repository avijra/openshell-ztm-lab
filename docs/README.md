# OpenShell on Zero Trust — Demo Lab

> **Securing AI Agents in Production with Zero Trust on ROSA**

Welcome to the hands-on demo lab for **NVIDIA OpenShell** running on Red Hat's **Zero Trust Validated Pattern (ZTVP)** on **ROSA** (Red Hat OpenShift Service on AWS).

## What You Will Learn

In this lab you will:

1. **Deploy** an AI agent sandbox from the OpenShift Developer Catalog — one click
2. **Run** the Pi coding agent powered by DeepSeek LLM inside a zero-trust sandbox
3. **Prove** that network access is restricted to only approved endpoints
4. **Prove** that filesystem access is locked down by Landlock LSM
5. **Observe** that API credentials are never visible inside the sandbox
6. **Explore** how Vault, SPIFFE, ACS, and External Secrets integrate

## The Story

AI agents are powerful — they can write code, call APIs, install packages, and interact with production systems. But with power comes risk:

- What if an agent exfiltrates data to an unauthorized server?
- What if it reads secrets from the Kubernetes API?
- What if it writes to system files or escalates privileges?

**Zero Trust for AI Agents** answers these questions. Every network connection, every file access, every credential usage is governed by explicit policy. Nothing is allowed by default.

## Lab Duration

| Section | Time |
|---------|------|
| Deploy from Catalog | 5 min |
| Client Setup | 2 min |
| AI Agent Coding | 5 min |
| Zero Trust Demos | 10 min |
| ZTVP Integration | 5 min |
| **Total** | **~30 min** |

## Fill In Your Environment

Use the form at the top of the page to enter your environment details. All commands in this lab will auto-populate with your values.

> **Tip:** Your instructor will provide the Bastion Host, SSH Password, Cluster Domain, and DeepSeek API Key.
