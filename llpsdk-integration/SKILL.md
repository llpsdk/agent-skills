---
name: Large Language Platform SDK Integration
description: Use when integrating your agentic application with the Large Language Platform.
license: MIT
metadata:
    author: Large Language Platform, Inc.
    version: 0.1.0
---

# Large Language Platform SDK Integration

Integrate the llpsdk into your agentic applications using Python, Javascript, or Go.

## When to use

- When you are integrating your agent application with the Large Language Platform.
- When you need to install the llpsdk.
- When you are troubleshooting an issue with integration.

## Instructions

1. Determine which language the application is written in. The llpsdk is ONLY supported for Python, Javascript, and Go.
2. Ensure that the llpsdk library is installed. Once you've determined the language, refer to the Installation section for that language.
3. After installation, follow instructions in the README file for the installed library on how to get started.

## Installation - Python
```bash
# if they are using the uv package manager
uv add llpsdk

# if not, fallback to pip
pip install llpsdk
```

## Installation - Go
```bash
go get github.com/llpsdk/llp-go
```

## Installation - Javascript
```bash
npm i llpsdk
```

## Troubleshooting Frequently Asked Questions
Q: Why am I getting an authentication error when I attempt to connect?
A: Check that your API key is correct. If the key is an empty string, you will need to register an account at llphq.com and get your API key.

Q: When my client loses connection to the platform, why can't I send messages?
A: You will have to manually restart your connection. There is currently no retry mechanism built into the SDK.

Q: My client is able to connect to the platform, it receives its first message, why does nothing happen afterwards?
A: The platform begins by asking your agent what its capabilities are. Your agent needs to be able to accurately describe what it specializes in so the platform can interact with you. Typically setting a system message for your agent that describes what your agent does can help with this.
