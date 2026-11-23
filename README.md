# Bomb98

**BOMB (1998 version).** Official homepage of BOMB, a French demoscene group (Skal, Made, Zappy, Patapom, Titan, Boz, Ben, Axel). It covers news (1998 demoparty results: Volcanic, LTP, Wired), BOMB memories (member pages), incoming projects, downloads (demos and intros), pixel art, links and credits.

Archive of an old static website, served as-is from bunny.net at the hostname
of [`site.json`](site.json). Kept online for history, no longer developed.

Full list of the sites: https://chanoj.github.io/

## Changes from the original site

The files are as served in 1998, apart from the changes needed to run in a modern browser:

- `index.htm` shows the intro page (`eye.htm`) in an iframe, in a window frame that follows the OS
  of the visitor on desktop and fills the screen in landscape on mobile (it used to open a popup).
  `?os=win10|win11|mac2012|mac2014|mac2020|mac2026` and `?device=mobile|desktop` force a look.
- The 2000-era `document.layers` scripts (menu rollovers, hover help, counter, gallery arrows, intro
  loader) use `getElementById`.
- The Java applets (`TmapCube`, `AnFade`) run with [CheerpJ](https://cheerpj.com/) (Community
  licence, runtime loaded from its CDN, credit on the credits page). Their `.class` files are patched:
  30 fps frame cap and click handler (a third-party ad dialog) disabled.
- `download/arclst.inc` is inlined into the `download/*.shtml` pages (no SSI on the static host).

## Layout

| Path | Content |
| --- | --- |
| `site/` | The website files, as served (from a local copy; hosting leftovers and working files removed) |
| `site.json` | The only site-specific settings: repository, storage zone, pull zone, hostname (plus zone ids, only until the first apply adopts them) |
| `infra/` | OpenTofu: storage zone, pull zone, hostname (state encrypted, committed) |
| `infra/secrets.env` | Proton Pass **references** to the secrets (vault `OldSites`), no secret values |
| `scripts/deploy.py` | Sync `site/` to the storage zone, purge the CDN |
| `scripts/smoke_test.py` | Check that the public site serves the files of `site/` |
| `scripts/bootstrap.sh` | Set up a new repository of the same kind (see below) |
| `scripts/import_storage.py` | One-time download of the storage zone into `site/` |

DNS is not managed here: the hostname is a CNAME to `<pull zone>.b-cdn.net`
at Infomaniak.

## Removal requests

Pages that list or describe people end with a small footer, "To be removed from this page, contact me." (a `mailto:` link). `scripts/add_removal_note.py` adds it to the pages it lists (idempotent; `--check` reports pages that lack it).

## Deploying

- **Website**: push a change under `site/` to `main` → workflow *Website: deploy*
  uploads the changed files, deletes the removed ones, purges the CDN and runs
  the smoke test. A manual run allows a dry run.
- **Infra**: pull requests touching `infra/` show the plan; apply is a manual
  run of *Infra: bunny.net* (action `apply`, from `main`), which commits the
  encrypted state.

GitHub secrets: `BUNNY_API_KEY` (bunny.net account API key) and
`TF_VAR_STATE_PASSPHRASE` (state encryption), both from Proton Pass vault
`OldSites`. The storage zone password and pull zone id are looked up from the
API key, never stored.

Locally (Proton Pass CLI logged in):

```bash
pass-cli run --env-file infra/secrets.env -- python3 scripts/deploy.py --dry-run
```

```bash
cd infra && pass-cli run --env-file secrets.env -- tofu plan
```

## Reuse for another site

The same files serve every old site of the bunny.net account; only `site.json`,
`site/`, the README title and the passphrase reference differ.

1. Copy the template into the new, empty repository folder (here `HotDIR97`):

   ```bash
   cd ~/Code/OrangeJuice97 && git ls-files | grep -v -e '^site/' -e '^site.json$' -e '^infra/terraform.tfstate$' | rsync -a --files-from=- . ~/Code/HotDIR97/
   ```

2. Fill in the site-specific parts, download the files and check everything
   (plan, deployment dry run, smoke test):

   ```bash
   cd ~/Code/HotDIR97 && git init -b main && scripts/bootstrap.sh init HotDIR97 chandra-sw chandra
   ```

3. Create the GitHub repository and its secrets, then push and run the
   *Infra: bunny.net* workflow with action `apply` (adopts the existing
   resources into the state):

   ```bash
   scripts/bootstrap.sh github
   ```

4. Once the first apply has adopted the resources, rename the zones after the
   repository (bunny.net cannot rename: the zones are replaced, new ones first):
   - at Infomaniak, point the CNAME of the hostname to `<repo, lowercase>.b-cdn.net`
     (any `*.b-cdn.net` name reaches bunny.net, which routes by hostname, so
     this is safe before and after the switch);
   - one pull request that removes the ids from `site.json` and sets
     `storage_zone` and `pullzone` to the lowercase repository name;
   - after merge, run *Infra: bunny.net* with action `apply`: it creates the
     new zones, moves the hostname, deletes the old zones, deploys `site/` to
     the new storage zone and runs the smoke test (a few minutes of errors at
     most, while the files are uploaded and the certificate issued).
