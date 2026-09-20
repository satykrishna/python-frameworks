# Python Frameworks for AI Models

A practical guide to connecting Python applications to hosted and local AI models.
The examples use the providers' official SDKs where possible and keep application code
portable by putting provider-specific setup behind a small function.

## What you need

- Python 3.10+
- A provider account and API key for hosted models, or [Ollama](https://ollama.com/)
  for local models
- A virtual environment for project dependencies

```bash
python -m venv .venv
source .venv/bin/activate       # Windows: .venv\Scripts\activate
python -m pip install --upgrade pip
```

Store credentials in environment variables rather than source code. A `.env` file is
convenient for local development, but do not commit it.

```bash
export OPENAI_API_KEY="..."
export ANTHROPIC_API_KEY="..."
export GOOGLE_API_KEY="..."
```

## The common pattern

Most model integrations have the same shape:

1. Install the provider SDK.
2. Set the provider's API key or endpoint.
3. Select a model by its provider-specific model ID.
4. Send messages or a prompt.
5. Read the returned text, tool call, or structured result.

Keep the model ID configurable so changing models does not require a code change:

```python
import os

MODEL = os.getenv("MODEL", "gpt-4o-mini")
```

## OpenAI

Install the official client:

```bash
python -m pip install openai
export OPENAI_API_KEY="..."
```

```python
from openai import OpenAI

client = OpenAI()  # Reads OPENAI_API_KEY automatically.

response = client.responses.create(
	model="gpt-4o-mini",
	input="Explain dependency injection in two sentences.",
)

print(response.output_text)
```

For a chat-style request, provide messages as the input:

```python
response = client.responses.create(
	model="gpt-4o-mini",
	input=[
		{"role": "system", "content": "You are a concise Python tutor."},
		{"role": "user", "content": "What is a context manager?"},
	],
)
```

## Anthropic Claude

Install the SDK and set its key:

```bash
python -m pip install anthropic
export ANTHROPIC_API_KEY="..."
```

```python
from anthropic import Anthropic

client = Anthropic()  # Reads ANTHROPIC_API_KEY automatically.

message = client.messages.create(
	model="claude-3-5-haiku-latest",
	max_tokens=256,
	system="You are a concise Python tutor.",
	messages=[
		{"role": "user", "content": "What is a context manager?"},
	],
)

print(message.content[0].text)
```

Anthropic separates the system prompt from the user and assistant messages, so keep
that distinction when moving an integration from another provider.

## Google Gemini

The current Google Gen AI SDK supports Gemini through the Gemini API and can also be
configured for Vertex AI.

```bash
python -m pip install google-genai
export GOOGLE_API_KEY="..."
```

```python
from google import genai

client = genai.Client()  # Reads GOOGLE_API_KEY automatically.

response = client.models.generate_content(
	model="gemini-2.0-flash",
	contents="Explain dependency injection in two sentences.",
)

print(response.text)
```

For Vertex AI, authenticate with Google Cloud and initialize the client with
`vertexai=True`, a project, and a location instead of using `GOOGLE_API_KEY`.

## Local models with Ollama

Ollama runs models on your machine and exposes a local HTTP API. Install Ollama,
download a model, and start it:

```bash
ollama pull llama3.2
python -m pip install ollama
```

```python
from ollama import chat

response = chat(
	model="llama3.2",
	messages=[
		{"role": "user", "content": "What is a context manager?"},
	],
)

print(response.message.content)
```

The default Ollama endpoint is `http://localhost:11434`. When the server runs
elsewhere, configure the SDK with `OLLAMA_HOST` or use its HTTP API directly.

## OpenAI-compatible endpoints

Many hosted and self-hosted services expose an OpenAI-compatible API. Reuse the
OpenAI SDK by changing `base_url` and the model ID:

```bash
python -m pip install openai
export AI_API_KEY="..."
```

```python
import os
from openai import OpenAI

client = OpenAI(
	api_key=os.environ["AI_API_KEY"],
	base_url="https://api.example.com/v1",
)

response = client.responses.create(
	model="provider-model-id",
	input="Hello from a portable Python client.",
)

print(response.output_text)
```

Check the service documentation before assuming full compatibility. Some endpoints
support chat completions but not the newer Responses API; in that case use
`client.chat.completions.create(...)` and read
`response.choices[0].message.content`.

## One application interface

When an application needs to switch providers, isolate the provider call behind a
small interface. The rest of the application should not know whether the response
came from a hosted or local model.

```python
from collections.abc import Callable


def ask_model(prompt: str, generate: Callable[[str], str]) -> str:
	"""Run application prompts through any configured model."""
	return generate(prompt)
```

Each provider adapter can then expose the same `str -> str` contract:

```python
def ask_openai(prompt: str) -> str:
	from openai import OpenAI

	response = OpenAI().responses.create(model="gpt-4o-mini", input=prompt)
	return response.output_text


def ask_ollama(prompt: str) -> str:
	from ollama import chat

	response = chat(model="llama3.2", messages=[{"role": "user", "content": prompt}])
	return response.message.content
```

For production code, expand this contract to include structured output, token usage,
latency, retries, and provider errors rather than hiding those details indefinitely.

## Choosing a connection

| Need | Good starting point | Connection |
| --- | --- | --- |
| Simple hosted API | OpenAI | `openai` SDK and `OPENAI_API_KEY` |
| Claude models | Anthropic | `anthropic` SDK and `ANTHROPIC_API_KEY` |
| Gemini models or Google Cloud | Google | `google-genai` SDK and API key or Vertex AI auth |
| Private or offline development | Ollama | Local HTTP service on port `11434` |
| A gateway or self-hosted server | OpenAI-compatible provider | OpenAI SDK with a custom `base_url` |

## Production checklist

- Keep API keys in environment variables or a secret manager.
- Pin SDK versions and model IDs in deployments.
- Set timeouts and retries with exponential backoff for transient failures.
- Log request IDs, latency, model ID, and token usage, but never log secrets or private prompts.
- Add rate-limit handling and application-level request budgets.
- Validate model output before using it in business logic.
- Use a small provider adapter so providers can be tested and replaced independently.
- Test with a cheap, deterministic model or a local model before running expensive integration tests.

## License

This repository is intended as a learning reference. Add the project's license here
when one is chosen.
