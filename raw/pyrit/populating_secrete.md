Populating Secrets
Microsoft AI Red Team
Contents
Creating Your .env File
What’s in .env_example?
What’s Next?
PyRIT loads API credentials from environment variables or .env files. This page shows how to set up your .env file with credentials for your AI provider.

Tip
For the full configuration story — including .env.local overrides, custom env file paths, and environment variable precedence — see the Configuration File guide.

Creating Your .env File
Tip
Using Docker? Create and edit ~/.pyrit/.env on your host machine — the Docker Compose setup automatically mounts it into the container. You don’t need to run these commands inside the container.

Create the PyRIT config directory and copy the example file:

mkdir -p ~/.pyrit
cp .env_example ~/.pyrit/.env

Edit ~/.pyrit/.env and fill in the credentials for your provider:

OpenAI
Azure OpenAI
Ollama (Local)
Groq
HuggingFace
OpenRouter
OPENAI_CHAT_ENDPOINT="https://api.openai.com/v1"
OPENAI_CHAT_KEY="sk-your-key-here"
OPENAI_CHAT_MODEL="gpt-4o"

Get your API key from platform.openai.com/api-keys.

Note
All these providers use the same three environment variables (OPENAI_CHAT_ENDPOINT, OPENAI_CHAT_KEY, OPENAI_CHAT_MODEL) because PyRIT’s OpenAIChatTarget works with any OpenAI-compatible API. Just point the endpoint to your provider and you’re set.

What’s in .env_example?
The .env_example file in the repository root contains entries for all supported targets — OpenAI chat, responses, realtime, image, TTS, video, Azure ML, embeddings, content safety, and more. Most users only need the three OPENAI_CHAT_* variables above. Fill in additional sections only as you need them.

What’s Next?
Configuration File (.pyrit_conf) — Set up the full configuration with initializers, database, and environment file management

Framework — Start using PyRIT
