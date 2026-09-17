# Co-Branding Open WebUI (PenguinTechnologies powered by Open WebUI)

Goal: add your own branding (name + logo) **alongside** Open WebUI's, without removing
or obscuring theirs — which keeps you inside the standard free license at any user count.

> Licensing note: everything here **adds** your branding and **leaves Open WebUI's mark
> visible**. That's the safe lane. The moment you delete/replace their logo you're into
> the branding clause (≤50 users, written permission, or enterprise license). Co-branding
> avoids that entirely.

---

## 0. Which install type do you have? (Read this first)

The rest of this guide has both **pip** and **Docker** paths. Know which you're on:

- **pip install** — you run it like `open-webui serve --port 3000` from inside a Python
  environment (e.g. a prompt showing `(openwebui)`). Files live inside the installed
  Python package. This is the setup on the current machine.
- **Docker** — you run it via `docker compose` / a `docker-compose.yml`. Files live in
  the container and are mounted via volumes.

The Docker sections (compose files, volume mounts) are for the Docker path. If you're on
pip, use **this section** for file locations and the env-var/CSS sections for the rest.

### Finding the pip package files (Windows)

With your environment activated, run:

```
python -c "import open_webui, os; print(os.path.dirname(open_webui.__file__))"
```

This prints the package root, e.g.:

```
C:\Users\Calculo\...\envs\openwebui\Lib\site-packages\open_webui
```

Inside that folder:
- `env.py` — holds the `WEBUI_NAME` logic and the auto-append " (Open WebUI)" behavior.
- `frontend\static\` — the logo, favicon, splash, and icon image assets. List them with:

```
dir "C:\Users\Calculo\...\site-packages\open_webui\frontend\static"
```

(use the actual path the first command printed).

### Setting the name on a pip install

Don't edit files — set the env var before serving, in the SAME prompt:

Command Prompt:
```
set WEBUI_NAME=PenguinTechnologies
open-webui serve --port 3000
```

PowerShell:
```
$env:WEBUI_NAME = "PenguinTechnologies"
open-webui serve --port 3000
```

Displays as `PenguinTechnologies (Open WebUI)` — the auto-append is your co-branding credit.

### Setting the logo on a pip install

Replace the PNGs in `frontend\static\` with your own, same names and dimensions.

> ⚠️ `pip install --upgrade open-webui` OVERWRITES these files. Keep your branding PNGs in
> a separate folder plus a tiny `.bat` script that re-copies them after every upgrade.
> (This is the pip equivalent of the Docker "updates revert my logo" problem.)

### Where the user DATA lives (not the same as the package)

Accounts, chats, settings, and the SQLite DB are stored separately from the code — in the
data directory, by default a `data\` folder under your working directory, or wherever the
`DATA_DIR` env var points. Don't confuse it with the package folder above: package = code
+ assets, data dir = state. Back this up before upgrades.

> For a REPEATABLE client offering, consider Docker instead of pip: the reverse-proxy
> pattern (Section 4) keeps branding outside the container so upgrades never touch it.
> pip is fine for a single PoC box but the "upgrades wipe the logo" chore multiplies
> across client machines.

---

## 1. Set the display name (supported, no code)

Set the `WEBUI_NAME` environment variable. Wrap it in quotes if it contains spaces.

In `docker-compose.yml`:

```yaml
services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    environment:
      WEBUI_NAME: "PenguinTechnologies"
      # ...your other env vars (model backend URL, etc.)
    ports:
      - "3000:8080"
    volumes:
      - open-webui:/app/backend/data
    restart: always

volumes:
  open-webui:
```

Behavior to know: if `WEBUI_NAME` is set to anything other than "Open WebUI", the app
**automatically appends " (Open WebUI)"** to the name — so it shows as
`PenguinTechnologies (Open WebUI)`. For co-branding this is actually a *feature*: it keeps
their attribution present for you automatically. Leave that behavior in place — it's the
cleanest possible "powered by" credit and requires zero extra work or licensing worry.

Restart after changing: `docker compose up -d`.

---

## 2. Add your logo

Two approaches, depending on how deep you want to go.

### Option A — Admin/theme + CSS (recommended, update-safe)

The logo appears in the sidebar and on the splash/loading screen. The least invasive way
to place yours is a **custom CSS override** that swaps the sidebar logo image for yours and
adds a "powered by Open WebUI" line, without editing app source files (which get wiped on
update).

1. Put your logo somewhere the browser can reach it — e.g. mount a file and serve it, or
   host it. A 200×60 PNG with transparency is the sweet spot for the sidebar.
2. In Open WebUI: **Admin Panel → Settings**, find the custom CSS / interface options, or
   inject CSS via a reverse proxy (see Option C). Example CSS:

```css
/* Replace the sidebar logo image with yours */
img[src*="logo"] {
  content: url("https://your-host/penguin-logo.png");
}

/* Add a small "powered by" credit under your branding */
.sidebar-footer::after {
  content: "Powered by Open WebUI";
  display: block;
  font-size: 11px;
  opacity: 0.6;
  text-align: center;
  padding: 4px 0;
}
```

(Selectors shift between releases — inspect the running UI with browser dev-tools to grab
the current class/element for the logo and footer, then target those.)

### Option B — Replace the static image files (more thorough, not update-safe)

Open WebUI ships logo/icon assets in the image directories, e.g.:
- `static/images/open-webui-logo.png`
- `static/images/open-webui-logo-transparent.png`
- `static/images/open-webui-icon.png`
- plus favicon / splash / Apple-touch-icon / manifest icons for full coverage

You can overwrite these with your own files of the same dimensions/names. Do it via a Docker
volume mount so an image update doesn't silently revert them, e.g.:

```yaml
    volumes:
      - ./branding/penguin-logo.png:/app/build/static/images/open-webui-logo.png:ro
      - ./branding/penguin-icon.png:/app/build/static/images/open-webui-icon.png:ro
```

> If you go this route for a **co-branding** setup, keep an Open WebUI mark visible
> somewhere (e.g. the auto-appended name, or the "powered by" CSS credit). Replacing every
> logo asset AND stripping the name is the white-label case that needs a license at 50+ users.

---

## 3. Favicon / browser tab title

The tab title follows `WEBUI_NAME`. Because of the auto-append behavior, it reads
`PenguinTechnologies (Open WebUI)` by default — again, fine and even helpful for
co-branding. Favicon lives among the static image assets in step 2B.

---

## 4. Serving a logo file + injecting CSS via reverse proxy (Option C)

If you don't want to touch app internals at all, put an Nginx (or Caddy/Traefik) reverse
proxy in front of Open WebUI and have it:
- serve your logo at a stable URL, and
- inject your custom CSS `<link>` into responses (sub_filter in Nginx).

This keeps ALL your branding outside the Open WebUI container, so app updates never touch
it. It's the most maintainable pattern for a repeatable client offering.

---

## Recommended setup for a client deployment

1. `WEBUI_NAME: "PenguinTechnologies"` — leave the auto-appended "(Open WebUI)" in place.
2. Custom CSS (via admin panel or reverse proxy) to place your logo in the sidebar + a
   small "Powered by Open WebUI" footer line.
3. Don't merge the two logos into one combined mark — keep them visually distinct
   (their brand guidelines ask for this).
4. Keep their icon/name present somewhere. You're adding, not removing.

Do that and no permission or enterprise license is needed regardless of each client's user
count, because Open WebUI's branding is still present — you've layered yours on top.

---

## When you'd still need to contact them (sales@openwebui.com)

- A client insists the Open WebUI mark be **completely gone** (true white-label), AND that
  deployment has **50+ users** in a rolling 30-day window.
- You want written cover for a repeatable, productized integrator offering.

Co-branding as described above avoids both.

*Not legal advice — the actual LICENSE file governs. Confirm your specific plan against it.*
