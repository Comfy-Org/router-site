# Build on Comfy Router

Comfy Router is one HTTP API at `https://api.comfy.org` for hosted image and video models, such as Seedream, Seedance, Kling, Nano Banana, FLUX, MiniMax, and Wan. You send the model's own JSON input and get the model's own output back. Each run draws Comfy Credits from the user's workspace.

The user may never have made an API key or run a paid API. Go one step at a time, say what each step does before you run it, and never spend credits the user has not agreed to.

## 1. Get a key
1. Check whether `COMFY_API_KEY` is set in the environment or in the project's `.env`. If it is, go to step 2.
2. If not, write `save_comfy_key.sh` in the project root (below) and run it with `sh save_comfy_key.sh`, with a timeout of at least 10 minutes (it waits for the user). In the same turn, tell the user:
   - A Comfy API key window is opening. If you don't have a key yet, click **Create a key via web**. Your browser opens the key page, and the window shows the steps and a box to paste the key.
   - Paste the key into the window, not into this chat.

   On macOS the script asks in a popup window, so the user never types a command. Elsewhere it asks in the terminal when it can.

   ```sh
   #!/bin/sh
   # Saves a Comfy API key to .env. The key goes from a password box straight to .env,
   # never to the terminal output or to your coding agent.
   LINK="https://platform.comfy.org/profile/api-keys?onboarding=router"
   cd "$(dirname "$0")" || exit 1

   if command -v osascript >/dev/null 2>&1; then
     # macOS: ask in a popup, since the agent's terminal can't take input.
     btn=$(osascript -e 'button returned of (display dialog "Comfy Router needs an API key. It starts with comfyui-." & return & return & "No key yet? Click Create a key via web." buttons {"Cancel", "Create a key via web", "I have a key"} default button "I have a key" with title "Comfy API key" with icon note)') || { echo "Canceled. No key saved."; exit 1; }
     if [ "$btn" = "Create a key via web" ]; then
       open "$LINK"
       msg='"Your browser opened the Comfy API keys page. Sign in if it asks." & return & return & "1. Enter a name for the key, such as My laptop." & return & "2. Click Generate key." & return & "3. Copy the key. It starts with comfyui-." & return & "4. Paste it below and click Save." & return & return & "The key stays hidden and is saved only to .env in your project."'
     else
       msg='"Paste your Comfy API key and click Save." & return & return & "The key stays hidden and is saved only to .env in your project."'
     fi
     key=$(osascript -e "text returned of (display dialog $msg default answer \"\" with hidden answer buttons {\"Cancel\", \"Save\"} default button \"Save\" with title \"Comfy API key\" with icon note)") || { echo "Canceled. No key saved."; exit 1; }
   elif [ -t 0 ]; then
     # A terminal that takes input (Linux, Windows Git Bash, or the user running this by hand).
     printf 'Did you create your Comfy API key at %s? [y/n] ' "$LINK"; read -r ans
     [ "$ans" = "y" ] || { echo "Create a key at $LINK, then run this again."; exit 1; }
     printf 'Paste your Comfy API key (it will not show): '; stty -echo; read -r key; stty echo; echo
   else
     echo "This terminal can't ask for the key. Run this yourself: sh \"$PWD/save_comfy_key.sh\""; exit 2
   fi

   case "$key" in
     comfyui-*) ;;
     *) echo "That doesn't look like a Comfy API key (it starts with comfyui-). No key saved. Run this again."; exit 1 ;;
   esac
   [ -f .env ] && grep -v '^COMFY_API_KEY=' .env > .env.tmp && mv .env.tmp .env
   printf 'COMFY_API_KEY=%s\n' "$key" >> .env
   grep -qx '.env' .gitignore 2>/dev/null || printf '\n.env\n' >> .gitignore
   echo "Saved COMFY_API_KEY to .env."
   ```

3. If the script prints "Run this yourself", your terminal can't ask for the key (Linux or Windows). Give the user that exact line to paste into their own terminal, and wait for them to say it printed "Saved".
4. If it prints "Canceled" or "doesn't look like a Comfy API key", tell the user and run it again when they're ready.
5. Once it prints "Saved", load `.env` and go on: `python-dotenv` in Python, `dotenv` in Node.js, or `set -a; . ./.env; set +a` before each `curl`. Never print or read back the key's value.

Keep the key on the server or in the local environment. Never put it in browser code, logs, commits, or your replies.

## 2. Pick how to call Router
Use cURL, as the steps below do. It needs nothing installed, and it matches the quickstart.

Use an SDK only when the user's project is already in that language:
- Python 3.10+: `python -m pip install "comfy-sdk>=0.3.0"`. `AsyncComfy()` reads `COMFY_API_KEY`. If the project's Python is older than 3.10 (`python3 --version`), stay on cURL and tell the user why. Don't install or upgrade Python for them.
- TypeScript, Node.js 22+: `npm install "@comfyorg/sdk@>=0.4.0"`. `comfy` reads `COMFY_API_KEY`.

## 3. Check the key (free)
List one model. It runs no model and spends no credits, so it works even when the workspace has no credits:

```bash
curl "https://api.comfy.org/v2/models?limit=1" -H "X-API-Key: $COMFY_API_KEY"
```

- `200` with a `data` list: the key works. Tell the user, then go to step 4.
- `401` (`error_type: unauthorized`): the key is missing or wrong. Go back to step 1. Do not go on.

## 4. Generate one image
Before you run it, tell the user this spends credits for one image, link the rate at https://docs.comfy.org/development/comfy-router/pricing, and wait for a yes.

Run Seedream 5.0 Flash, `byteplus/seedream-5-0-flash-260915`, with `{"prompt": "A red fox trotting through a snowy pine forest, cinematic lighting", "size": "1024x1024"}`. With cURL (or the SDK's `client.models.run`):

```bash
curl https://api.comfy.org/v2/models/byteplus/seedream-5-0-flash-260915 \
  -H "X-API-Key: $COMFY_API_KEY" -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{"prompt": "A red fox trotting through a snowy pine forest, cinematic lighting", "size": "1024x1024"}' \
  -D headers.txt -o response.json
```

- Success: the image link is `data[0].url`. Download the file right away: the provider clears the link after 24 hours. Show the user where it saved, and report `X-Comfy-Credits-Used` from the response headers (credits; 211 credits is 1 US dollar).
- Timeout or a dropped connection: don't run it again on your own. A second run with a new `Idempotency-Key` bills again, so ask the user first.
- `402` (`error_type: insufficient_credits`): the key works, but the workspace has no credits. Router charged nothing. Tell the user to add credits at https://platform.comfy.org, and stop until they say they have. Do not debug the request body: Router checks credits first.
- `422`: the body failed the model's schema. Each entry in `detail` names a field (`loc`) and what is wrong (`msg`). Fix it and run again.

## 5. Offer a video (only if the user wants it)
Stop after the image. Ask whether they want to try a video, and give them the Seedance 2.5 rate from the pricing page first. Do nothing more unless they say yes.

If they say yes, use the queue, because a video takes minutes. Start from the image you just made, as the first frame:
1. Submit to `POST https://api.comfy.org/v2/models/byteplus/dreamina-seedance-2-5-260628/requests` with an `Idempotency-Key: $(uuidgen)` header (the SDK's `client.models.submit` makes one). The body:

   ```json
   {
     "content": [
       {"type": "text", "text": "The fox turns and runs deeper into the forest"},
       {"type": "image_url", "image_url": {"url": "<the image's data[0].url>"}, "role": "first_frame"}
     ],
     "duration": 5, "ratio": "adaptive", "resolution": "720p"
   }
   ```
2. Poll `.../requests/<request_id>/status` until it completes, waiting the `Retry-After` the server names each time.
3. Collect the result from `.../requests/<request_id>`, and download the video.

Read the queue page (from `llms.txt`) before you change the queue flow.

## Read the docs before you guess
Fetch https://docs.comfy.org/llms.txt. It links:
- `https://docs.comfy.org/_llms/en/comfy-router.txt`: every Router page, for the queue, errors, uploads, and billing.
- `https://docs.comfy.org/router-schemas/<provider>/<model>.json`: each model's input schema. Read it before you write a request body.
- `https://docs.comfy.org/router-openapi.yaml`: the full API.

## Find other models
`GET https://api.comfy.org/v2/models` lists every model Router can run (page with `next_cursor`). Never guess a field: read the model's schema first.
