# Committing the handoff to GitHub

No installation, no command line. This is all done in the browser.

---

## What you are doing

Putting the `handoff` folder into `luigimf/castillians-engine` so your engineers can reach the
prototype and specs from the same place as the code.

**A note on the word "commit":** in GitHub, saving a change *is* committing it. You are not
publishing or deploying anything — you are adding files and writing a one-line note about what
you added. Nothing breaks, and every version is kept, so anything can be undone later.

---

## Before you start

Unzip the handoff download somewhere you can find it — your Desktop is fine. You should have a
folder called `handoff` containing `SD-3453`, `SD-3455`, `SD-3456` and a `README.md`.

---

## Step 1 — Open the repository

Go to **https://github.com/luigimf/castillians-engine**

Since it is empty, you will see a mostly blank page with some setup instructions. Ignore those.

---

## Step 2 — Start an upload

Look for the **Add file** button near the top right, and choose **Upload files**.

If the repo is completely empty you may instead see a link that reads *"uploading an existing
file"* in the setup text. Either one takes you to the same place.

---

## Step 3 — Drag the folder in

Drag your `handoff` folder onto the upload area.

GitHub keeps the folder structure, so everything lands in the right place. It will take a
moment — the prototype files are around 2 MB each.

Wait until every file shows as uploaded before continuing.

---

## Step 4 — Describe what you added

Below the upload area there are two boxes. In the first, short one, write:

```
Add Engineer Dashboard design handoff and prototype
```

Leave the second box empty, and leave **"Commit directly to the main branch"** selected.

---

## Step 5 — Save

Click the green **Commit changes** button.

Done. The files are in the repository.

---

## Step 6 — Share it with your engineers

Send them this link:

```
https://github.com/luigimf/castillians-engine/tree/main/handoff
```

Point them at `handoff/README.md` first — it explains what each bundle contains and the order
to build in.

### If they want to open the prototype

GitHub will not run an HTML file from that page — it shows the code instead. They should click
the file, then click the **Download** button (or **Raw**, then save the page), and open the
saved file in a browser.

**Better option, if you want a link that just opens:** turn on GitHub Pages.

1. In the repo, click **Settings**
2. Choose **Pages** in the left sidebar
3. Under **Source**, pick **Deploy from a branch**
4. Select branch **main**, folder **/ (root)**, then **Save**

After a minute or two the prototype is live at:

```
https://luigimf.github.io/castillians-engine/handoff/prototype-standalone.html
```

Anyone with that link can open it in a browser. Note this makes it publicly reachable by anyone
who has the URL, so skip this step if the content is sensitive.

---

## Updating it later

When I send a new version of a bundle, repeat steps 2 to 5 with the new files. GitHub notices
they already exist and replaces them, keeping the old versions in the history.

---

## If something goes wrong

**"I can't find the Add file button."** You may not be signed in, or may not have write access
to the repository. Check you are signed in as the account that owns it.

**"The upload failed."** Individual files must be under 25 MB through the browser. Everything
here is well under that, so if it fails it is usually a connection drop — refresh and try again.

**"I uploaded it to the wrong place."** Nothing is lost. Tell me what happened and I will talk
you through moving it.

**"I want to undo it."** Every version is kept permanently. Tell me and I will walk you back.
