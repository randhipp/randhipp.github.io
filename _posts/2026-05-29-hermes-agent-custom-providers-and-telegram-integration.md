---
layout: post
title: "Power Your AI Assistant: Configuring Custom LLMs and Telegram with Hermes Agent"
date: 2026-05-29 12:00:00 +0700
categories: [AI, Automation, DevTools]
tags: [hermes-agent, llm, telegram, open-source]
---

Hermes Agent is an open-source AI agent framework by Nous Research that runs in your terminal, messaging platforms, and IDEs. It belongs to the same category as Claude Code or Codex—autonomous coding and task-execution agents that use tool calling to interact with your system.

What makes Hermes different is its ability to connect to virtually any LLM provider and operate across multiple platforms. Whether you want to spin up a private model on your own GPU cluster or chat with your assistant from Telegram on the go, Hermes handles it.

In this guide, we’ll walk through how to set up a **custom LLM endpoint** using the built-in CLI and how to bridge your agent to **Telegram**.

## Part 1: Setting Up Custom Providers via CLI

One of Hermes' strongest features is being provider-agnostic. While it supports major services like Anthropic, OpenRouter, and OpenAI out-of-the-box, developers often need to connect to custom endpoints—for example, self-hosted models via Ollama, vLLM, or private inference servers.

Instead of manually editing configuration files, Hermes provides an interactive CLI tool called `hermes model` to handle provider selection and custom endpoint registration.

### Step-by-Step Custom Provider Setup

Run the following command in your terminal:

```bash
hermes model
```

You will be greeted with a menu listing supported providers. To add a custom endpoint, follow these steps:

1.  **Select "Custom Endpoint"**: When the list appears, scroll down and select the option for a custom or unsupported provider.
2.  **Enter Base URL**: You will be prompted for the base URL of your inference server. 
    *   *Example:* `http://localhost:8000/v1` or `https://api.my-private-provider.com/inference`
    *   *Note:* Most modern inference engines (like Ollama or vLLM) are compatible if they expose the standard `/v1` OpenAI-compatible path.
3.  **Provide API Key**: If your endpoint requires authentication, enter your API key when prompted. If the connection is unauthenticated (common for local setups), you can skip this.
4.  **Specify Model Name**: Enter the name of the model hosted at that endpoint (e.g., `llama-3.1-70b` or `mistral-large`).

Once configured, Hermes adds this custom provider to your environment. You can switch between it and your default public providers instantly without restarting the process.

## Part 2: Integrating with Telegram

Hermes includes a multi-platform gateway that allows your agent to receive messages and execute tasks on various chat apps, including Telegram. This effectively puts a personal AI assistant with full tool access (file editing, shell commands, web search) right in your pocket.

### Step 1: Create a Telegram Bot

1.  Open Telegram and search for **@BotFather**.
2.  Send the command `/newbot`.
3.  Follow the prompts to name your bot and choose a username (ending in `bot`).
4.  Once created, BotFather will give you an **API Token**. Copy this token; you'll need it to connect the two services.

### Step 2: Configure Hermes

You don't need to hunt through config files. Use the setup wizard:

```bash
hermes gateway setup
```

1.  When prompted, select **Telegram**.
2.  Paste the **API Token** from BotFather.
3.  Confirm the settings.

Alternatively, you can export the token into your environment before starting the gateway:

```bash
export TELEGRAM_BOT_TOKEN="YOUR_TOKEN_HERE"
```

### Step 3: Start the Gateway

Now that the connection is established, launch the gateway:

```bash
# Start in the foreground (for testing)
hermes gateway run

# Install as a background service (for production)
hermes gateway install
systemctl --user start hermes-gateway
```

### Getting Started on Telegram

Open Telegram, search for your new bot, and send a message. Hermes will now respond. 

Because Hermes has full tool capabilities, you aren't limited to simple conversation. You can ask the Telegram bot to:
*   Run shell commands and check system stats.
*   Read and edit files on your machine.
*   Schedule recurring jobs (cron) that deliver their results back to your chat.
*   Analyze images or browse the web.

### Pro Tip: Voice Messages

If you have the faster-whisper library installed (`pip install faster-whisper`), Hermes automatically transcribes voice notes sent from Telegram so you can communicate hands-free.
