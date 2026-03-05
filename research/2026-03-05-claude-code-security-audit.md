# Using Claude Code for MacOS Security Audits Without Breaking Things

> **Executive Summary:** AI coding agents like Claude Code can be powerful tools for security auditing, but their capability is exactly what makes them dangerous if misused. This guide establishes a safe, practical methodology for using Claude Code to audit your Mac's security posture — maximizing insight while minimizing risk.

**Date:** 2026-03-05  
**Category:** Technology / Security  
**Tags:** Claude Code, macOS, security audit, AI agents, cybersecurity

---

## The Problem

You have a powerful AI coding agent that can read files, execute shell commands, and make system changes. You want to use it to analyze your Mac's security. But:

- It could run destructive commands
- It might expose sensitive data in its context
- Broad prompts invite broad (risky) actions
- "Make my system secure" is an invitation for disaster

The solution isn't to avoid using these tools — it's to use them with discipline.

---

## Core Principles

### 1. Read-Only First, Always

Every security audit task should start with explicit constraints:

> "Analyze and report — don't make changes."

Claude Code should examine your system, not modify it. Modifications come later, after you've reviewed findings and decided what to change yourself.

### 2. Scope Tightly

The tighter your prompt, the safer the execution.

| ❌ Bad Prompt | ✅ Good Prompt |
|---------------|----------------|
| "Make my Mac secure" | "Audit my SSH configuration and list weaknesses" |
| "Fix my firewall" | "Check if the macOS firewall is enabled and report its settings" |
| "Harden my system" | "List all Login Items and LaunchAgents with their paths" |

Broad prompts invite broad actions. Narrow prompts produce focused, reviewable output.

### 3. Respect the Permission System

Claude Code prompts before executing commands by default. This is your safety net.

**Never use `--dangerously-skip-permissions` for security work.**

Every command it wants to run should appear on your screen for approval. Read them. Understand them. Reject anything that looks excessive.

### 4. No Sudo Without Scrutiny

If Claude Code requests `sudo` access, that's a yellow flag. Ask yourself:

- Is elevated access actually necessary for this specific check?
- What exactly will this command do?
- Could this be accomplished without root?

Most read-only audits don't need sudo. When they do, review the exact command first.

---

## Safe Audit Tasks

These tasks are read-only and low-risk. They gather information without modifying your system.

### System Security Configuration

```
"Check if FileVault disk encryption is enabled"
"Check if the macOS firewall is enabled and show its configuration"
"Check System Integrity Protection (SIP) status"
"Check if Gatekeeper is enabled"
```

### Login and Persistence

```
"List all Login Items for my user"
"List all LaunchAgents in ~/Library/LaunchAgents and /Library/LaunchAgents"
"List all LaunchDaemons in /Library/LaunchDaemons"
"Show any cron jobs configured for my user"
```

### Privacy Permissions Audit

```
"List which apps have Full Disk Access"
"List which apps have Accessibility permissions"
"List which apps have Screen Recording permissions"
"List which apps have Input Monitoring permissions"
```

### Network Exposure

```
"List all listening ports and the processes that own them"
"Review my SSH config (~/.ssh/config) for security issues"
"Check if SSH is accepting remote connections"
"List active network connections"
```

### Software Hygiene

```
"List installed Homebrew packages and flag any that are significantly outdated"
"Check for Homebrew packages with known security advisories"
"List browser extensions installed in Chrome/Firefox/Safari"
"Show what software has automatic update checking disabled"
```

### File Permissions

```
"Find world-writable files in /usr/local"
"List files with setuid or setgid bits set"
"Check permissions on my ~/.ssh directory and files"
"Find any .env or config files with overly permissive permissions"
```

---

## What to Avoid

### Active Scanning

Don't ask Claude Code to run `nmap`, `masscan`, or similar active network scanners. These:

- Generate traffic that can trigger ISP alerts
- May violate terms of service
- Can disrupt network devices
- Are overkill for a personal security audit

### Automatic Remediation

Don't ask Claude Code to "fix" things automatically. Phrases to avoid:

- "Fix any security issues you find"
- "Harden my system based on best practices"
- "Update my firewall rules"

Instead: "Report your findings. I'll decide what to change."

### Credential Exposure

Be aware that anything Claude Code reads enters its context window. Don't ask it to:

- Read your password manager database
- Analyze files containing API keys (unless necessary)
- Parse your keychain

If you must analyze sensitive files, consider redacting them first.

### Kernel and Low-Level Changes

Stay away from:

- Kernel extension analysis or modification
- Boot configuration changes  
- System file modifications

These require expertise beyond what an AI audit should attempt.

---

## Recommended Workflow

### Step 1: Start with Explicit Constraints

Begin your Claude Code session with a clear, bounded prompt:

```
"I want to audit my Mac's security posture. 

Rules:
- Report findings only — don't make any changes
- Ask before running any command that requires elevated permissions
- If you need sudo for something, explain why first
- Output findings to a markdown file

Start by checking: FileVault status, Firewall status, SIP status, and Gatekeeper status."
```

### Step 2: Review Its Plan

Before executing, Claude Code should show you what it intends to run. Review each command. If something looks off, ask for clarification or reject it.

### Step 3: Progress Through Categories

Work through audit categories one at a time:

1. System configuration (FileVault, Firewall, SIP, Gatekeeper)
2. Persistence mechanisms (Login Items, LaunchAgents)
3. Privacy permissions
4. Network exposure
5. Software hygiene
6. File permissions

### Step 4: Get a Written Report

Ask Claude Code to compile findings into a structured markdown file:

```
"Compile all findings into a security audit report. Format:
- Category
- Finding
- Risk level (Low/Medium/High)
- Recommendation

Save to ~/Desktop/security-audit-2026-03-05.md"
```

### Step 5: Make Changes Yourself

Review the report offline. Decide what to address. Either:

- Make changes manually
- Ask Claude Code to propose specific changes for your approval (one at a time)

Never batch-approve remediation.

---

## Example Audit Prompt

Here's a complete prompt you can use to start a security audit session:

```
I want you to audit my Mac's security configuration. 

Ground rules:
1. This is a READ-ONLY audit. Don't modify anything.
2. Ask me before running any command requiring sudo.
3. If a check fails or requires special access, note it and move on.
4. Compile findings into a markdown report at the end.

Audit these areas:
- Disk encryption (FileVault)
- Firewall status and configuration
- System Integrity Protection
- Gatekeeper
- Login Items and LaunchAgents
- Apps with sensitive privacy permissions (Full Disk, Accessibility, Screen Recording)
- Listening network ports
- SSH configuration
- Outdated Homebrew packages
- File permission issues in common locations

For each finding, rate risk as Low/Medium/High and provide a one-line recommendation.

Start now. Show me the commands you'll run for the first category before executing.
```

---

## Limitations

This methodology is for **personal security hygiene**, not:

- Penetration testing
- Malware forensics
- Enterprise security compliance
- Incident response

For those use cases, use purpose-built tools and (ideally) professional expertise.

---

## Conclusion

Claude Code is a force multiplier for security auditing — it can rapidly check configurations you might forget to examine, and it can synthesize findings into actionable reports. But its power requires discipline.

The formula is simple:

**Tight scope + read-only by default + human review = safe, useful audits**

Don't let the AI drive. Let it investigate while you steer.

---

## Sources

- Apple Platform Security Guide (Tier S)
- macOS Security Compliance Project (Tier A)
- Anthropic Claude Code Documentation (Tier A)
- CIS Apple macOS Benchmarks (Tier A)
