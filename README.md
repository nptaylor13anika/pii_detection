```python
#!/usr/bin/env python3
"""
stream_chat_select.py – stream Llamafile / OpenAI-compatible SSE responses
without long hangs on Ctrl+C (works on Linux, macOS, Windows).

Usage example:
    python stream_chat_select.py "Hello there!" "How are you?"
"""

import json
import select
import socket
import sys
import time
import urllib.error
import urllib.request

BASE_URL = "http://localhost:11434"          # ← change if needed
READ_SLICE_SEC = 0.10                        # poll interval (100 ms)


def stream_chat(messages, *, model="llama-3-8b", **extra):
    """
    Send POST /v1/chat/completions with stream=True and print tokens
    as they arrive.  Returns the full reply string (already printed).

    KeyboardInterrupt will stop *immediately* (≤ READ_SLICE_SEC).
    """
    payload = {
        "model": model,
        "stream": True,
        "messages": messages,
        **extra,
    }
    full_reply_tokens = []

    try:
        req = urllib.request.Request(
            f"{BASE_URL}/v1/chat/completions",
            data=json.dumps(payload).encode(),
            headers={
                "Content-Type": "application/json",
                "Accept": "text/event-stream",
            },
            method="POST",
        )

        # ❶ Establish TCP & read HTTP headers (can still take timeout seconds)
        with urllib.request.urlopen(req, timeout=60) as resp:
            # ❷ Grab the raw socket and make it non-blocking
            sock: socket.socket = resp.fp.raw._sock
            sock.setblocking(False)

            buf = b""
            while True:
                # ❸ Wait up to READ_SLICE_SEC for bytes *or* a signal
                rlist, _, _ = select.select([sock], [], [], READ_SLICE_SEC)

                if rlist:
                    chunk = sock.recv(64 * 1024)   # read whatever’s ready
                    if not chunk:                  # EOF
                        break
                    buf += chunk

                # ❹ Process complete CRLF-terminated SSE lines in buf
                while b"\r\n" in buf:
                    line, buf = buf.split(b"\r\n", 1)

                    if not line.startswith(b"data: "):
                        continue
                    if line.strip() == b"data: [DONE]":
                        raise StopIteration

                    payload = json.loads(line[6:])   # strip "data: "
                    token = payload["choices"][0]["delta"].get("content", "")
                    if token:
                        print(token, end="", flush=True)
                        full_reply_tokens.append(token)

    except urllib.error.HTTPError as e:
        sys.stderr.write(e.read().decode() or str(e))
        raise
    except StopIteration:
        pass      # graceful stream end
    except KeyboardInterrupt:
        print("\n\n[Interrupted by user]\n")
        # let caller know it was interrupted (optional: return partial)
    finally:
        print()    # trailing newline so next prompt is on a new line

    return "".join(full_reply_tokens)


# --------------------------------------------------------------------------- #
# Small demo: loop over prompts supplied on the command line
# --------------------------------------------------------------------------- #

if __name__ == "__main__":
    prompts = sys.argv[1:] or ["Hello!", "Tell me a joke about cats."]
    role_system = {"role": "system", "content": "You are a helpful assistant."}

    for user_prompt in prompts:
        msgs = [
            role_system,
            {"role": "user", "content": user_prompt},
        ]
        print(f"\n>>> {user_prompt}")
        try:
            reply = stream_chat(msgs, temperature=0.7)
            print(f"[full reply length: {len(reply)} chars]")
        except KeyboardInterrupt:
            # Give the user an obvious escape hatch between generations
            print("Stopping further generations.")
            break
```
