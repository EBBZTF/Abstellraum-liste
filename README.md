# Abstellraum

A searchable inventory of the cellar boxes, served from the Raspberry Pi on the
home network. One NFC tag on the cupboard door opens it; the tags on the boxes
themselves stay as they are.

The spreadsheet **is** the database. `boxes.xlsx` is read on demand: when its
modification time changes, the page is rebuilt on the next request. Editing the
sheet never requires a restart, and between edits the rendered page is cached
and served in about a millisecond.

- **Page:** http://192.168.1.149/
- **Pi:** `berdi-websites@192.168.1.149`, app on `127.0.0.1:8091`, Caddy in front on port 80
- **Checkout on the Pi:** `/home/berdi-websites/abstellraum`, a clone of this repo
- **Deploy:** push to GitHub, then pull on the Pi

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
   git push
   ```

4. Pull it onto the Pi — one line from the laptop:

   ```sh
   ssh berdi-websites@192.168.1.149 'cd ~/abstellraum && git pull --ff-only'
   ```

That is the whole loop for a spreadsheet edit. **No restart:** the app checks the
file's timestamp on every request and re-reads it when it moves. Pull the page
down to refresh on the phone and the change is there.

`--ff-only` matters. Nothing on the Pi writes to tracked files, so a pull should
always fast-forward; if it ever cannot, something is wrong and you want it to
stop rather than quietly commit a merge on the Pi.

Only if you changed **`boxes.py`** does anything need restarting:

```sh
ssh berdi-websites@192.168.1.149 'cd ~/abstellraum && git pull --ff-only && sudo systemctl restart boxes'
```

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
git push
ssh berdi-websites@192.168.1.149 'cd ~/abstellraum && git pull --ff-only'
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

**2. Clone the repo**

```sh
git clone https://github.com/EBBZTF/Abstellraum-liste.git ~/abstellraum
```

The path matters: `boxes.service` expects `/home/berdi-websites/abstellraum`.
Clone it elsewhere and you have to edit the three paths in that file.

If the GitHub repo is private, clone over SSH instead and add the Pi's public
key as a read-only deploy key (**Settings → Deploy keys**), otherwise every
`git pull` will ask for a token:

```sh
ssh-keygen -t ed25519 -C "pi-abstellraum" -f ~/.ssh/id_ed25519 -N ""
cat ~/.ssh/id_ed25519.pub        # paste into GitHub
git clone git@github.com:EBBZTF/Abstellraum-liste.git ~/abstellraum
```

**3. Python environment**

```sh
cd ~/abstellraum
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
```

Check it serves before wiring up systemd:

```sh
.venv/bin/python boxes.py     # then: curl -s localhost:8091 | head -c 200
```

**4. The service**

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

**5. Optional: push-to-deploy instead of pull**

Not needed for the clone-and-pull setup above — skip this unless you want
pushes from the laptop to deploy by themselves.

`deploy/post-receive` is a git hook for a **bare** repo on the Pi. It checks the
pushed commit out and restarts the service only when `boxes.py` changed. To use
it instead of pulling, create a bare repo alongside the working tree, install
the hook into it, and push there from the laptop:

```sh
git init --bare ~/abstellraum.git
install -m 755 ~/abstellraum/deploy/post-receive ~/abstellraum.git/hooks/post-receive
```

```sh
# on the laptop
git remote add pi berdi-websites@192.168.1.149:/home/berdi-websites/abstellraum.git
git push pi main
```

For the hook to restart the service without a password:

```sh
sudo visudo -f /etc/sudoers.d/boxes-restart
```

one line, permitting that command and nothing else:

```
berdi-websites ALL=(root) NOPASSWD: /usr/bin/systemctl restart boxes
```

Without it the push still deploys; it just prints the restart command for you
to run.

**6. Pin the Pi's address**

Confirm what it actually is — do not trust this README:

```sh
hostname -I        # first value is the LAN address
```

The NFC tag hard-codes that address, so give the Pi a DHCP reservation in the
router and it will never move. Without one, the tag stops working some day for
no visible reason, and the failure looks exactly like the app being broken.

If the address ever does change, the tag is the only thing that needs
rewriting — the app, the service and the Caddy block are all address-independent.

---

## Caddy on the Pi

Caddy on this Pi is **not** a normal web server on port 80. It sits behind a
Cloudflare tunnel: `auto_https off` globally, bound to `127.0.0.1:8080`, serving
`berdi-racing.com` by way of `cloudflared`. Nothing on the home network can
reach it, by design, and nothing listens on port 80 at all.

So the inventory needs its own listener. **Append** this repo's `Caddyfile` to
`/etc/caddy/Caddyfile` — never overwrite it, that file is what keeps
berdi-racing.com up:

```sh
sudo nano /etc/caddy/Caddyfile        # paste the block at the very end
sudo caddy validate --config /etc/caddy/Caddyfile
sudo systemctl reload caddy
```

It is a `:80` block, so it cannot collide with the existing `:8080` ones. The
global `auto_https off` is what lets a bare `:80` work without Caddy going off
to fetch a certificate for it.

**The page is restricted to the home network by Caddy itself.** The block only
serves requests whose source address is in a private range; everything else gets
a bare 404. So even if a port-80 forward is still sitting in the router from the
pre-tunnel days, the inventory is not reachable from the internet.

That check uses `remote_ip`, the real TCP source address — deliberately not
`client_ip`, which reads `X-Forwarded-For` and can be forged by whoever sends
the request.

Then, on the Pi:

```sh
curl -s http://localhost/ | grep -o 'class="box"' | wc -l    # expect 98
```

98 is 92 boxes plus 6 open shelves. `validate` only proves the file parses — this
proves port 80 actually reaches the app.

To confirm it really is closed from outside: find your public address with
`curl -s https://api.ipify.org`, then open `http://<that address>/` on a phone
**with Wi-Fi turned off**. You should get nothing. With Wi-Fi back on,
`http://192.168.1.149/` gives you the inventory.

A stray port-80 forward in the router is still worth deleting if you find one —
the tunnel does not use it — but it is no longer what stands between the cellar
list and the internet.

The block deliberately has no `bind 192.168.1.149`. Restricting the listener to
the LAN interface sounds tighter but buys nothing here: a router forward
delivers to exactly that address anyway, so it would not stop one. What it would
do is bake the IP into Caddy, and if the address ever changed the listener would
fail and Caddy would not start — taking berdi-racing.com down with it.

### If you would rather not touch Caddy

Reasonable, given what that config is holding up. Skip Caddy entirely, put the
app on the network directly, and let the tag carry the port:

```
Environment=BOXES_HOST=0.0.0.0
```

in `boxes.service`, then `sudo systemctl daemon-reload && sudo systemctl restart
boxes`, and write the tag as `http://192.168.1.149:8091/`.

The app sends `Cache-Control: no-store` itself — Caddy repeating it was
belt-and-braces, not a dependency — and the port only ever mattered for a URL
somebody types, which is not what a tag is for.

**But this route has no source-address check.** `0.0.0.0` means the app answers
on every interface to anyone who can reach it, and waitress has no equivalent of
the `remote_ip` matcher. Then the router really is the only thing keeping the
inventory off the internet, and a stray forward on 8091 would expose it. If you
take this route, confirm there is no forward rather than assuming.

## First-time setup — on the laptop

In this folder:

```sh
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
```

`origin` already points at GitHub, which is where the Pi pulls from — there is
no second remote to add.

Run it locally any time:

```sh
.venv/bin/python boxes.py        # http://127.0.0.1:8091/
```

An SSH key saves typing a password on every deploy:

```sh
ssh-copy-id berdi-websites@192.168.1.149
```

---

## Writing the NFC tag

The NTAG215 tags hold far more than a URL, so there is plenty of room.

1. Install **NFC Tools** (free, App Store) on an iPhone.
2. **Write → Add a record → URL/URI**.
3. Enter the address exactly, with the scheme and the trailing slash:

   ```
   http://192.168.1.149/
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
| `Caddyfile` | A `:80` block to **append** to the Pi's existing Caddy config, not replace it |
| `boxes.service` | systemd unit |
| `deploy/post-receive` | Git hook for push-to-deploy. Unused by the current pull-based setup; see step 5 |
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
