# Pickup

A small libadwaita/GTK4 dashboard for picking up where you left off with a
coding agent. It lists your [Claude Code](https://claude.com/claude-code),
[pi](https://pi.dev) and [OpenCode](https://opencode.ai) sessions together,
newest first, showing each session's title, agent and working directory. One
click resumes a session in a terminal, and you can start a new named session
in any directory with any of them.

Each row carries a badge naming its agent. When you have sessions from more
than one, a dropdown inside the search box narrows the list to one of them.

## Run

```bash
python3 pickup.py
```

Requires PyGObject with GTK 4 and libadwaita (`python3-gobject`, `gtk4`,
`libadwaita` — already present on most GNOME systems).

To install it as a launchable app, put the script on your `PATH` and copy the
desktop file:

```bash
install -Dm755 pickup.py ~/.local/bin/pickup
cp io.github.jerem.Pickup.desktop ~/.local/share/applications/
```

## Flatpak

Build and install with `flatpak-builder` (needs `org.gnome.Sdk//50`):

```bash
flatpak-builder --user --install --force-clean build-dir \
    io.github.jerem.Pickup.yaml
flatpak run io.github.jerem.Pickup
```

The sandbox does not protect you from the app. Its whole job is to open a
terminal on the host and run an agent there, and the only way a Flatpak can do
that is `--talk-name=org.freedesktop.Flatpak`, which lets it run any command on
the host through `flatpak-spawn --host`. Treat it as you would the same script
installed outside Flatpak.

What the file grants do limit is what the app's own code reads and writes
directly:

- `--filesystem=~/.claude/projects` holds Claude Code's session logs. The app
  reads them, and appends an entry when you rename or colour a session. The
  rest of `~/.claude`, including Claude Code's credentials, is not mounted.
- `--filesystem=~/.pi/agent/sessions` is the same for pi. The rest of `~/.pi`,
  including its `auth.json`, is not mounted.
- `--filesystem=xdg-data/opencode:ro` holds OpenCode's session database,
  mounted read-only: renames and deletes go through OpenCode's own CLI on the
  host instead. That database also holds OpenCode's credentials, so the app
  can read them. It never does, but the grant can't be narrower than the file.

Everything else goes through `flatpak-spawn --host`: launching your terminal
and the agent, `mkdir -p` for a new session's location, `xdg-open` for a
project folder, `systemctl --user show-environment` to find the ssh-agent, and
`gio trash` for deleted sessions, and `opencode` to rename or delete an
OpenCode session. Terminals open as host processes, so they see
your real files without the app needing broad home access. Starred sessions
and the parse cache are kept in the app's own directories.

One wrinkle worth knowing about: the file chooser portal doesn't hand a
sandboxed app the folder you picked. It exports it through the Documents portal
and returns a `/run/user/UID/doc/DOCID/name` mount instead — a path that means
nothing to the terminal being launched, and granting `--filesystem` doesn't
change that. The portal's own `Info` method refuses to answer inside the sandbox
("Not allowed in sandbox"), so `host_path()` resolves the id back to a real path
via `flatpak documents` on the host.

## How it works

- **Discovery** — scans `~/.claude/projects/*/*.jsonl` and
  `~/.pi/agent/sessions/*/*.jsonl`. Each `.jsonl` is one session. For each, it
  reads the working directory, the title, and the file's modification time,
  then sorts newest first. A Claude Code title is the `/rename` name, then the
  AI-generated title, then the first prompt. A pi title is the `/name` name,
  then the first prompt. Lines are read as bytes and substring-tested before
  being parsed, so the big assistant payloads are never decoded. OpenCode
  sessions come from `$XDG_DATA_HOME/opencode/opencode.db`, opened read-only:
  the title, directory and last update of each top-level session, and its
  user turns for search. Subagent and archived sessions are left out, as
  `opencode session list` does. After a
  project moves, Claude Code can leave a second, metadata-only file with the
  same session id in the old project's directory. The two are shown as one
  row, pointing at the file that holds the conversation.
- **The index** — logs get large (a long session can reach hundreds of
  megabytes), and re-reading them all on every scan does not scale. Parsed state
  is cached per file in `$XDG_CACHE_HOME/pickup/index.json`,
  including the byte offset parsing stopped at. Bumping its version discards
  what is cached, which is how a new field such as the session colour gets
  picked up for logs already indexed. Session logs are append-only, so
  an unchanged file costs a `stat()` and a grown one costs only its new bytes —
  a file that gained 2 KB is 2 KB of reading, not 350 MB. A file that shrank was
  rewritten rather than appended to, so it is parsed again from the top. Only
  the first scan pays full price; delete the file to force one.
- **Search** — the box matches every whitespace-separated term (AND) against
  the title, working directory, session id, *and the conversation content*, so
  you can find a session by something you typed in it. A term that matches
  nothing verbatim is treated as a typo: it's corrected against the words the
  sessions actually contain (Damerau-Levenshtein — one edit for terms under 8
  characters, two beyond, none under 4, and a swapped pair of letters counts as
  one), then searched as if spelled correctly. So `sesion` and `cluade` find
  what `session` and `claude` find. Correctly spelled queries are unaffected —
  the typo pass only ever runs for a term that was about to match nothing.
- **New session** — the *+* button (or <kbd>Ctrl</kbd>+<kbd>N</kbd>) asks for a
  name and a location, then runs `claude --name <name>` there, or
  `pi --name <name>` if you pick pi in the dialog's Agent choice, or plain
  `opencode`, which has no such flag and titles the session itself. The Agent
  choice lists the agents installed on the host, and is left out when Claude
  Code is the only one. The name doubles
  as the folder to create: typing *“Partner Dashboard”* with `~/dev` as the
  location gives `~/dev/partner-dashboard`, kept in sync in the field as you
  type, so you always see where you'll land. Editing the location by hand stops
  that and leaves whatever you typed alone; browsing for a folder sets a new
  parent and re-appends. Anything missing is created (with parents), so a project
  and its first session start in one step, and the dialog reopens at the parent
  so the next session lands beside the last rather than inside it. For Claude
  Code and pi the name is the agent's own `--name`, so it shows up inside the
  agent too.
- **Resume** — clicking *Resume* (or a row) opens your default terminal in the
  session's working directory and runs `claude --resume <session-id>`, or
  `pi --session <file>` for a pi session, or `opencode --session <id>` for an
  OpenCode one. Pi only looks an id up among the current project's sessions,
  so it gets the file instead. The shell stays
  open after the agent exits.
- **Open folder** — *⋮ → Open folder* shows the session's working directory in
  your file manager. Like everything else that touches a project directory, the
  launch happens on the host: the sandbox can't see the folder, and the OpenURI
  portal wants a file descriptor for a directory we have no way to open from in
  here. Falls back to `gio open` if `xdg-open` isn't installed, and says so if
  the directory is gone.
- **Colour** — a session coloured with `/color` gets a bar of that colour down
  the leading edge of its row, and the swatches at the top of *⋮* set one from
  here. The bar is a background layer on the row, so colouring a session never
  shifts its title out of line with the rows around it. It is not a border: a
  border is painted outside the row's background, so at the rounded corner a
  boxed list gives its first and last row it separates from the row and leaves
  a gap. A background stays inside that corner and follows it. Like the
  rename, it appends the same `agent-color` entry Claude Code writes, so the
  colour also applies inside Claude Code. Pi has no session colour, so for a
  pi session the app appends a `custom` entry of its own (`pickup:color`),
  which pi keeps out of the model's context. OpenCode has none either, and
  its database is not the app's to write to, so an OpenCode session's colour
  is kept in the app's config directory. The eraser swatch clears it, which is
  what `/color default` does. The colour name is searchable, so typing `pink`
  finds the pink ones.
- **Star** — the star toggle moves a session into a **Favorites** section shown
  above the rest (each section ordered most-recent first). Stars persist in the
  app's config directory.
- **Rename** — *⋮ → Rename…* sets a custom name by appending the same
  `custom-title` entry that `/rename` writes, so the name also shows up inside
  Claude Code. For a pi session it appends the `session_info` entry `/name`
  writes, attached to the session's latest entry the way pi does it. An
  OpenCode session is renamed through OpenCode itself, with
  `opencode api session.update`, so its server stays the only writer.
- **Delete** — *⋮ → Delete* asks for confirmation, then moves the session's
  `.jsonl` file to the trash (recoverable from your file manager). OpenCode
  has no trash, so an OpenCode session is deleted for good with `opencode
  session delete`, along with its subagent sessions, and the dialog says so.
- **Live updates** — a `Gio.FileMonitor` on each session root, and on the
  directory holding OpenCode's database (whose writes land in its `-wal`
  file), watches for
  changes (debounced ~800 ms). New sessions, renames, deletions, and activity
  bumps are reconciled *in place* — only the affected rows change, so your
  scroll position and search text are preserved. This fires on every write to
  the session you are sitting in, which is what makes the index matter: a
  rescan re-reads only what actually changed. The manual refresh button forces
  an immediate re-scan.

### Which terminal opens

It honours your system default, in this order:

1. `$TERMINAL` environment variable
2. `xdg-terminal-exec` (the freedesktop default-terminal standard)
3. the first installed terminal it recognises — ghostty, ptyxis, gnome-console
   (kgx), gnome-terminal, konsole, kitty, alacritty, foot, wezterm, xterm

Set `TERMINAL=ghostty` (or whatever you use) to force a specific one.

### The ssh-agent

The command each terminal runs starts by exporting `SSH_AUTH_SOCK`, so `git
push` over SSH works there without a passphrase prompt. That has to be done
explicitly, because three separate layers drop the variable. Flatpak does not
put it in the sandbox, `flatpak-spawn` gives the host command the sandbox's
environment where it was never set, and a login shell on the host does not set
it either, since the desktop session keeps it in the systemd user environment
rather than in profile files. The value is read back from `systemctl --user
show-environment`.

It is exported inside the command rather than passed as `flatpak-spawn --env`.
Terminals that run as a single-instance daemon hand the command to an existing
process, and an environment requested at spawn time never arrives. If the app
already has an agent of its own, which is the case when it runs outside
Flatpak, nothing is added. If no agent is found, the command is unchanged.
