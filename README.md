# Ai7 Marketplace

Skill packs for Ai7 (`hz.ai7.work`). The Ai7 server lists every pack in this repo in the
Skills dialog (Browse) and installs them on demand.

## Layout

```
.ai7-plugin/marketplace.json        # the index Ai7 reads; one entry per pack
packs/<pack-name>/
  plugin.json                       # agent-plugins manifest (name, version, description, author, license)
  .ai7-plugin/plugin.json           # browse details: displayName, category, logo (file in the same dir)
  logo-*.svg                        # pack logo (svg/png/jpg/webp)
  skills/<skill-name>/SKILL.md      # YAML frontmatter (name + description) + skill body
```

## Adding a pack

1. `mkdir -p packs/<pack-name>/skills/<skill-name>` and add `SKILL.md` with frontmatter:

   ```markdown
   ---
   name: <skill-name>
   description: One or two sentences saying what it does and when to use it.
   ---

   The skill body…
   ```

2. Add `plugin.json` (copy an existing pack, change `name`/`description`), and
   `.ai7-plugin/plugin.json` with the display name and logo.
3. Add an entry to `.ai7-plugin/marketplace.json` with `"source": "./packs/<pack-name>"`.
4. Commit and push to `main`. The server picks up changes on its next catalog refresh
   (10-minute cache).

Rules: public repo (fetched anonymously), text companion files only
(`.md .txt .json .yaml .csv`), each skill ≤100 KB, ≤40 companion files ≤400 KB total.

## Push-size rule

Preview reads GitHub's commit API for HEAD with a 512 KB cap, and that response embeds the
full patch of every file the commit touches. Keep each push's total added/changed content
under ~400 KB — split a large pack across several pushes — or installs fail with
"Could not preview a supported plugin from this repository" until a smaller commit lands.
