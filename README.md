# GitHub → LLM → Slack Commit Summarizer

An **automated n8n-powered bot** that listens for GitHub commits, extracts the commit message and lines changed, runs them through an **LLM** for natural-language summarization, and posts the result directly to a Slack channel.  
Perfect for keeping your team updated with **human-friendly commit summaries** — without reading raw diffs.

---

## Features

- **GitHub Webhook Listener**  
  Captures push events automatically.
- **Diff Extraction**  
  Gathers commit messages and the number of lines changed.
- **LLM Processing**  
  Sends commit details to an LLM (e.g., OpenAI, Gemini, DeepSeek) for a natural, concise summary.
- **Slack Posting**  
  Sends the LLM’s summary to a chosen Slack channel with formatting.
- **Easy to Customize**  
  Swap out the LLM provider, adjust message format, or filter commits.

---

## Tech Stack

- **[n8n](https://n8n.io/)** – Workflow automation
- **GitHub Webhooks** – Event source
- **Slack API** – Message delivery
- **LLM API** – Summarization engine

---

## Prerequisites

Before you start, you’ll need:

- **n8n** installed locally or hosted (Docker recommended)
- **GitHub Personal Access Token** (for private repo access if needed)
- **Slack Bot Token** with `chat:write` scope
- **LLM API Key** (OpenAI, Gemini, DeepSeek, etc.)

---

## Setup Guide

### 1. Clone This Repo
```bash
git clone https://github.com/Abdullah-Khan-Sherwani/Project-Updates-Bot.git
cd Project-Updates-Bot
