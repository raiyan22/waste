Great. Since you're authorized to use all four, we can set this up as a local failover proxy on Windows.

One thing first: there are currently two OpenCode config generations in the docs. The stable/current OpenCode docs use provider, while the OpenCode 2 beta docs use providers. 
O
OpenCode
+1

I'll assume you're using the normal opencode CLI, not opencode2.

1. Install Python

In PowerShell:

python --version


If that prints something like:

Python 3.12.x


you're good.

If it says Python isn't recognized, install Python from:

Python.org

During installation, make sure "Add Python to PATH" is checked.

2. Create the folder

Run:

mkdir "$env:USERPROFILE\opencode-key-rotator"
cd "$env:USERPROFILE\opencode-key-rotator"


You'll have:

C:\Users\YOURNAME\opencode-key-rotator\

3. Put your keys.md there

Copy your existing file:

copy "C:\PATH\TO\keys.md" "$env:USERPROFILE\opencode-key-rotator\keys.md"


For example, if it's on your Desktop:

copy "$env:USERPROFILE\Desktop\keys.md" "$env:USERPROFILE\opencode-key-rotator\keys.md"


Make the file look like:

sk-or-v1-xxxxxxxxxxxxxxxx
sk-or-v1-yyyyyyyyyyyyyyyy
sk-or-v1-zzzzzzzzzzzzzzzz
sk-or-v1-wwwwwwwwwwwwwwww


One key per line.

Don't put the real keys in this chat.

4. Create a Python virtual environment

Run:

cd "$env:USERPROFILE\opencode-key-rotator"

python -m venv .venv


Activate it:

.\.venv\Scripts\Activate.ps1


If PowerShell complains about script execution, run:

Set-ExecutionPolicy -Scope CurrentUser RemoteSigned


Then:

.\.venv\Scripts\Activate.ps1


You should see something like:

(.venv) PS C:\Users\You\opencode-key-rotator>

5. Install the proxy dependencies

Run:

pip install fastapi uvicorn httpx


Then create:

notepad proxy.py


Paste this:

import asyncio
import re
from pathlib import Path

import httpx
import uvicorn
from fastapi import FastAPI, Request, Response
from fastapi.responses import StreamingResponse


HOST = "127.0.0.1"
PORT = 8787

UPSTREAM = "https://openrouter.ai/api/v1"

KEYS_FILE = (
    Path.home()
    / "opencode-key-rotator"
    / "keys.md"
)

app = FastAPI()

key_lock = asyncio.Lock()
current_key = 0


def load_keys():
    if not KEYS_FILE.exists():
        raise RuntimeError(
            f"Keys file not found: {KEYS_FILE}"
        )

    text = KEYS_FILE.read_text(
        encoding="utf-8"
    )

    keys = re.findall(
        r"sk-or-[A-Za-z0-9_-]+",
        text
    )

    # Remove duplicates while preserving order.
    keys = list(dict.fromkeys(keys))

    if not keys:
        raise RuntimeError(
            f"No OpenRouter keys found in {KEYS_FILE}"
        )

    return keys


def get_current_index():
    return current_key


async def set_current_index(index):
    global current_key

    async with key_lock:
        current_key = index


@app.get("/health")
async def health():
    keys = load_keys()

    return {
        "status": "ok",
        "keys_loaded": len(keys),
        "current_key": current_key % len(keys),
    }


@app.api_route(
    "/v1/{path:path}",
    methods=[
        "GET",
        "POST",
        "PUT",
        "PATCH",
        "DELETE",
        "OPTIONS",
    ],
)
async def proxy(path: str, request: Request):

    body = await request.body()

    keys = load_keys()

    async with key_lock:
        start_index = current_key % len(keys)

    for attempt in range(len(keys)):

        key_index = (
            start_index + attempt
        ) % len(keys)

        key = keys[key_index]

        url = f"{UPSTREAM}/{path}"

        headers = {
            "Authorization": f"Bearer {key}",
            "Content-Type": "application/json",
        }

        accept = request.headers.get("accept")

        if accept:
            headers["Accept"] = accept

        client = httpx.AsyncClient(
            timeout=httpx.Timeout(
                connect=30,
                read=300,
                write=300,
                pool=30,
            ),
            follow_redirects=True,
        )

        try:
            upstream_request = client.build_request(
                request.method,
                url,
                headers=headers,
                content=body,
                params=request.query_params,
            )

            upstream = await client.send(
                upstream_request,
                stream=True,
            )

        except Exception as e:

            await client.aclose()

            if attempt == len(keys) - 1:
                return Response(
                    content=f"Upstream connection error: {e}",
                    status_code=502,
                )

            continue

        # Rate limited:
        # try the next authorized key.
        if upstream.status_code == 429:

            await upstream.aread()
            await upstream.aclose()
            await client.aclose()

            next_index = (
                key_index + 1
            ) % len(keys)

            await set_current_index(
                next_index
            )

            print(
                f"[ROTATE] Key {key_index + 1} "
                f"received 429 -> "
                f"switching to key {next_index + 1}"
            )

            continue

        # Non-429 response.
        await set_current_index(
            key_index
        )

        async def stream():

            try:
                async for chunk in upstream.aiter_raw():
                    yield chunk

            finally:
                await upstream.aclose()
                await client.aclose()

        response_headers = {}

        for name, value in upstream.headers.items():

            if name.lower() in {
                "content-length",
                "transfer-encoding",
                "connection",
                "content-encoding",
            }:
                continue

            response_headers[name] = value

        return StreamingResponse(
            stream(),
            status_code=upstream.status_code,
            headers=response_headers,
            media_type=upstream.headers.get(
                "content-type",
                "application/json",
            ),
        )

    return Response(
        content="All configured OpenRouter keys returned 429.",
        status_code=429,
    )


if __name__ == "__main__":

    print(
        f"OpenRouter key rotator listening on "
        f"http://{HOST}:{PORT}"
    )

    print(
        f"Keys file: {KEYS_FILE}"
    )

    print(
        f"Loaded keys: {len(load_keys())}"
    )

    uvicorn.run(
        app,
        host=HOST,
        port=PORT,
    )


Save and close Notepad.

6. Test that your four keys are detected

Make sure the virtual environment is still active:

.\.venv\Scripts\Activate.ps1


Then:

python proxy.py


You should see:

OpenRouter key rotator listening on http://127.0.0.1:8787
Keys file: C:\Users\YOURNAME\opencode-key-rotator\keys.md
Loaded keys: 4


If it says Loaded keys: 4, that's good.

Leave this PowerShell window open.

7. Test the proxy

Open a second PowerShell window.

Run:

curl.exe http://127.0.0.1:8787/health


You should get:

{"status":"ok","keys_loaded":4,"current_key":0}


Now test OpenRouter:

curl.exe http://127.0.0.1:8787/v1/models


You should get OpenRouter's model list.

At this point:

OpenCode
   ↓
127.0.0.1:8787
   ↓
OpenRouter
   ↓
key 1


is working.

8. Configure OpenCode

Now we need to tell OpenCode to use your local proxy.

First find your OpenCode config.

The easiest way is to create/edit the project-level opencode.json in the directory where you normally run OpenCode.

For example:

cd C:\Users\YOURNAME\your-project
notepad opencode.json


Use this:

{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "openrouter-rotator": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "OpenRouter 4-Key Rotator",
      "options": {
        "baseURL": "http://127.0.0.1:8787/v1",
        "apiKey": "local"
      },
      "models": {
        "YOUR_MODEL_ID": {
          "name": "Your Model"
        }
      }
    }
  }
}


This is the stable OpenCode configuration format documented currently. OpenCode explicitly supports custom providers using @ai-sdk/openai-compatible and custom baseURL endpoints. 
O
OpenCode

9. Replace YOUR_MODEL_ID

This is important.

For example, if you're currently using:

anthropic/claude-sonnet-4.5


then use:

{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "openrouter-rotator": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "OpenRouter 4-Key Rotator",
      "options": {
        "baseURL": "http://127.0.0.1:8787/v1",
        "apiKey": "local"
      },
      "models": {
        "anthropic/claude-sonnet-4.5": {
          "name": "Claude Sonnet"
        }
      }
    }
  }
}


If you don't know your model ID, run:

curl.exe http://127.0.0.1:8787/v1/models


and look at the "id" fields.

10. Start OpenCode

Keep the proxy window running.

In your project directory:

opencode


Inside OpenCode:

/models


You should see:

OpenRouter 4-Key Rotator


and your configured model.

OpenCode supports custom provider endpoints specifically for proxy/custom-endpoint use cases. 
O
OpenCode

11. How the rotation works

Normally:

OpenCode
    ↓
localhost:8787
    ↓
OpenRouter
    ↓
Key 1


If OpenRouter returns HTTP 429:

Key 1 → 429
       ↓
Key 2


If key 2 also returns 429:

Key 2 → 429
       ↓
Key 3


Then:

Key 3 → 429
       ↓
Key 4


So you don't need to log in or log out of OpenCode at all.

The proxy handles the credentials; OpenCode only knows about:

http://127.0.0.1:8787/v1

12. Your final folder

You should have:

C:\Users\YOURNAME\opencode-key-rotator\
│
├── keys.md
├── proxy.py
└── .venv\


And your OpenCode project has:

your-project\
└── opencode.json


Do not put keys.md into your Git repository. Add it to .gitignore if necessary:

keys.md


Also don't commit the actual API keys anywhere.

One improvement I'd make

The script above rotates on 429, but for a real OpenCode workload I'd make it a little smarter: temporarily mark a rate-limited key as unavailable for a cooldown period, rather than immediately using it again on the next request. That prevents this pattern:

key1 → 429
key2 → 429
key3 → OK
key3 → OK
key1 → 429 again
key2 → 429 again


Instead, it could maintain:

key 1  🔴 cooldown
key 2  🔴 cooldown
key 3  🟢 available
key 4  🟢 available


and automatically bring keys back after their Retry-After/cooldown expires.

That would be the version I'd actually recommend for continuous OpenCode use. OpenRouter exposes rate-limit responses/headers, so the proxy can use those rather than blindly hammering a limited key.

O
Sources
