# Abstellraum

A searchable inventory of the cellar boxes, served from the Raspberry Pi on the
home network. One NFC tag on the cupboard door opens it; the tags on the boxes
themselves stay as they are.

The spreadsheet **is** the database. `boxes.xlsx` is read on demand: when its
modification time changes, the page is rebuilt on the next request. Editing the
sheet never requires a restart, and between edits the rendered page is cached
and served in about a millisecond.

- **Page:** http://192.168.1.50/
- **Pi:** `berdi-websites@192.168.1.50`, app on `127.0.0.1:8091`, Caddy in front on port 80
- **Working tree on the Pi:** `/home/berdi-websites/abstellraum`
- **Bare repo on the Pi:** `/home/berdi-websites/abstellraum.git`

---

## Everyday: change something in the cellar

1. Open `boxes.xlsx`, edit, save. Keep the layout: a new box starts on the row
   where you put a number in column **B**; every row under it with something in
   column **D** belongs to that box. Column **A** is the shelf and only needs
   filling on the box's first row.
2. Check it locally before it goes live:

   ```sh
   .venv/bin/python boxes.py
   ```

   then open http://127.0.0.1:8091/ and search for what you changed. Ctrl-C to stop.
3. Ship it:

   ```sh
   git add boxes.xlsx
   git commit -m "Kiste 37: Stoffe raus, Vorhänge rein"
   git push pi main
   ```

The push prints what it did. For a spreadsheet-only change you will see:

```
  boxes.py unchanged — no restart needed
  the app picks the new spreadsheet up on the next request
```

That is correct — there is nothing else to do. Pull the page down to refresh on
the phone and the change is there.

If you changed `boxes.py` as well, the push restarts the service and says so.

---

## Undo a bad spreadsheet edit

`.xlsx` is a zip file, so git stores each version whole. That makes rollback a
matter of picking a version, not merging one.

**Saved a mess but not committed yet** — throw the working copy away:

```sh
git checkout -- boxes.xlsx
```

**Already committed and pushed** — find the version you want and bring it back:

```sh
git log --oneline -- boxes.xlsx        # every version of the sheet
git checkout <sha> -- boxes.xlsx       # restore the sheet as of that commit
git commit -m "Tabelle zurück auf <sha>"
git push pi main
```

**Not sure which version you want** — open an old one without touching the
current file:

```sh
git show <sha>:boxes.xlsx > /tmp/alt.xlsx && open /tmp/alt.xlsx
```

**Two machines edited the sheet and git reports a conflict** — a binary file
cannot be merged, so choose one side outright, then redo the other edit by hand:

```sh
git checkout --ours boxes.xlsx     # keep the version you just made
git checkout --theirs boxes.xlsx   # or keep the version already on the Pi
git add boxes.xlsx && git commit
```

---

## First-time setup — on the Pi

Everything here runs as `berdi-websites`. Nothing in this repo needs to run as
root except installing the service unit and touching Caddy's config.

**1. Packages**

```sh
sudo apt update && sudo apt install -y git python3-venv
```

**2. Bare repo, working tree, and the deploy hook**

```sh
mkdir -p ~/abstellraum
git init --bare ~/abstellraum.git
```

The hook is in this repo but has to be copied into the bare repo by hand the
first time — afterwards it updates itself only if you copy it again, so if you
ever change `deploy/post-receive`, repeat this step.

```sh
# easiest: paste the file's contents, or scp it over from the laptop first
install -m 755 /path/to/post-receive ~/abstellraum.git/hooks/post-receive
```

**3. Push from the laptop now** (see the laptop section below). The hook checks
the files out into `~/abstellraum`. It will try to restart a service that does
not exist yet and print the manual command — that is expected at this point.

**4. Python environment**

```sh
cd ~/abstellraum
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
```

Check it serves before wiring up systemd:

```sh
.venv/bin/python boxes.py     # then: curl -s localhost:8091 | head -c 200
```

**5. The service**

```sh
sudo cp ~/abstellraum/boxes.service /etc/systemd/system/boxes.service
sudo systemctl daemon-reload
sudo systemctl enable --now boxes
systemctl status boxes
```

Logs, when something looks wrong:

```sh
journalctl -u boxes -f
```

**6. Let the hook restart the service without a password**

Without this the deploy still works; it just prints the restart command for you
to run. To make code pushes fully automatic:

```sh
sudo visudo -f /etc/sudoers.d/boxes-restart
```

and put in exactly this one line:

```
berdi-websites ALL=(root) NOPASSWD: /usr/bin/systemctl restart boxes
```

It permits that one command and nothing else.

**7. Pin the Pi's address**

The NFC tag will contain `192.168.1.50`. Give the Pi a DHCP reservation in the
router so that address never moves — otherwise the tag stops working one day
for no visible reason.

---

## Caddy on the Pi

Caddy is already running and already has a config. **Do not overwrite
`/etc/caddy/Caddyfile` with the `Caddyfile` in this repo.** Read what is there
first:

```sh
cat /etc/caddy/Caddyfile
```

Then merge, depending on what you find:

- **Nothing is using port 80** — append this repo's `Caddyfile` to
  `/etc/caddy/Caddyfile`.
- **A `:80 { … }` block already exists** — do not add a second one, Caddy will
  refuse to start. Add the two lines inside the existing block instead:

  ```
  reverse_proxy 127.0.0.1:8091
  header Cache-Control "no-store"
  ```

  If that block already serves something else, give the inventory its own path
  and put the path in the tag instead of the bare address:

  ```
  handle_path /abstellraum* {
      reverse_proxy 127.0.0.1:8091
      header Cache-Control "no-store"
  }
  ```

Port 8080 on loopback is Caddy's own; the app deliberately uses 8091.

Check and reload — `validate` first, so a typo cannot take the running server
down with it:

```sh
sudo caddy validate --config /etc/caddy/Caddyfile
sudo systemctl reload caddy
```

`Cache-Control: no-store` matters more than it looks. Safari will otherwise
show a cached inventory, and a list that confidently gives the wrong box number
is worse than one that takes a moment to load.

---

## First-time setup — on the laptop

In this folder:

```sh
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
git remote add pi berdi-websites@192.168.1.50:/home/berdi-websites/abstellraum.git
```

Run it locally any time:

```sh
.venv/bin/python boxes.py        # http://127.0.0.1:8091/
```

An SSH key saves typing a password on every push:

```sh
ssh-copy-id berdi-websites@192.168.1.50
```

---

## Writing the NFC tag

The NTAG215 tags hold far more than a URL, so there is plenty of room.

1. Install **NFC Tools** (free, App Store) on an iPhone.
2. **Write → Add a record → URL/URI**.
3. Enter the address exactly, with the scheme and the trailing slash:

   ```
   http://192.168.1.50/
   ```

   Use `http://`, not `https://` — there is no certificate on the Pi and Safari
   will refuse the connection if you ask for TLS.
4. **Write / Write to the tag**, then hold the tag against the top of the phone
   until it confirms.
5. Stick it on the cupboard door and test: lock the phone but leave the screen
   on, hold it to the tag, and a banner should appear that opens the page.

Background tag reading works on iPhone XS and newer. The screen has to be on
and the phone unlocked-or-lock-screen — it does not work while the phone is in
a pocket, which is the behaviour you want for a tag on a door.

**Do not lock the tag.** Locking is permanent, and if the Pi's address ever
changes you would have to throw the tag away and write a new one.

This only works on the home network. Away from home the address does not
resolve to anything.

---

## Configuration

Set in `boxes.service`; all three have defaults, so the app also just runs.

| Variable | Default | Meaning |
|---|---|---|
| `BOXES_SHEET` | `boxes.xlsx` next to `boxes.py` | Which spreadsheet to read |
| `BOXES_HOST` | `127.0.0.1` | Loopback only — Caddy is the front door |
| `BOXES_PORT` | `8091` | 8080 is taken by Caddy |

To reach the app directly from another machine while debugging, start it with
`BOXES_HOST=0.0.0.0` — as a one-off on the command line, not in the unit file.

---

## What's in here

| File | |
|---|---|
| `boxes.py` | The whole app: reads the sheet, renders one page, serves it with waitress |
| `boxes.xlsx` | The inventory. The `Inhalt` sheet is what gets published |
| `Caddyfile` | Reverse proxy to merge into the Pi's existing Caddy config |
| `boxes.service` | systemd unit |
| `deploy/post-receive` | Git hook for the bare repo on the Pi |
| `.gitattributes` | Marks `.xlsx` binary so git never diffs or merges it |

### How the spreadsheet is read

From the `Inhalt` sheet, starting at row 3 — rows 1 and 2 are the title and the
header:

| Column | | |
|---|---|---|
| **A** | `Regal` | The shelf. Set once per box, on its first row; merged down the rest |
| **B** | `Box` | The box number. A value here starts a new box |
| **C** | `►` | Decorative bullet, ignored |
| **D** | `Inhalt` | One content item per row |

Currently 92 boxes and 227 items, 14 of the boxes empty. Empty boxes stay in the
list marked *leer*, so you can see at a glance which ones are free.

The `Regale (2)` sheet is published underneath as **Offene Regale**, one entry
per column. The other sheets — `Beschriftung`, `Regale`, `Kisten`,
`Kisten hinten` — are for printing labels and are not published.

Shelf numbers render as `Regal 2`; `RK` and `B` are printed as they stand. Box
27 has no shelf in the sheet and shows as *ohne Regal* — fill column A on its
row if you want that fixed.

Box numbers written as formulas (`=B13+1`) are read from the cached result.

### Searching

The search box filters as you type, in the browser, with no request to the Pi.
It matches against everything: contents, box numbers, shelf names, and the word
*leer*. Matching boxes open by themselves and the matched words are highlighted,
so when the hit is on something the collapsed row does not show, you can still
see what matched.

Umlauts fold both ways — `muesli` finds *Müsli*, and `aufbewahrungsglaeser`
finds *Aufbewahrungs-gläser* despite the hyphen. Tapping a box open closes
whichever was open before, so the list never runs away down the screen.
