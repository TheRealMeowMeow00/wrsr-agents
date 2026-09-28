# wrsr-agents

Knowledge for AI coding agents working on mods for **Workers & Resources:
Soviet Republic**, packaged first of all for Claude.

An agent pays for every fact it has to rediscover: reading dozens of game
files, disassembling the executable, repeating a mistake somebody already
debugged. This kit writes those facts down once, with their sources.

## Install

### Claude Code (recommended)

In a Claude Code session:

```
/plugin marketplace add TheRealMeowMeow00/wrsr-agents
/plugin install wrsr@wrsr-agents
```

or from a shell:

```
claude plugin marketplace add TheRealMeowMeow00/wrsr-agents
claude plugin install wrsr@wrsr-agents
```

That is all. The `wrsr-modding` skill loads by itself whenever a task is about
the game, in any folder; `/wrsr:wrsr-modding` loads it by hand. In every other
session only its one-paragraph description is present.

Updates: turn on auto-update for the `wrsr-agents` marketplace under
**Marketplaces** in `/plugin`, or run `/plugin marketplace update wrsr-agents`
now and then.

### Claude apps

Download `wrsr-modding.skill` from the releases page and add it as a custom
skill.

### Without the plugin system

Copy `skills/wrsr-modding/` into `~/.claude/skills/` for every project, or into
a project's `.claude/skills/`.

### Other agents

`AGENTS.md` points them to the same files: keep the repository next to your
mods, or copy `AGENTS.md` and `skills/` into a project. Keep them out of a
Workshop item folder, because the whole item folder is uploaded.

## What is inside

- `skills/wrsr-modding/SKILL.md`: the core, loaded when a task is about WRSR.
  What the game is, the rules that prevent the known failures, where to look,
  community etiquette. About 2 000 tokens.
- `skills/wrsr-modding/references/`: the details, one file per kind of task,
  opened only when the task needs it.

| File | Covers |
|---|---|
| `workshop.md` | Workshop items: `workshopconfig.ini`, item types, layouts, Steam limits, packaging plugins |
| `buildings.md` | cloning a building, `building.ini`, `renderconfig.ini`, materials, failure symptoms |
| `vehicles.md` | `script.ini` tokens, rail specifics, track speed limits |
| `native-mods.md` | TesmioLoader and Republic Mod Loader, writing and shipping a plugin, Linux |
| `reverse-engineering.md` | identifying the build, techniques, porting, signatures, patching, crashes |
| `internals.md` | engine structures: resources, buildings, electricity, citizens, walking, clocks, deposits, rail, language files |
| `pitfalls.md` | what already went wrong for somebody, and how it looked |

- `.claude-plugin/`: the plugin and marketplace manifests.
- `AGENTS.md`: a pointer for agents other than Claude.

## About cost

Only the skill's description, about 200 tokens, is present in every session.
The core loads when a task is about the game, and a reference file only when
the task needs it. Prompt caching is per account, so nobody shares a cache with
anyone else; the real saving is the exploration an agent does not have to
repeat.

## Contributing

- Facts only, each traceable to a game file, a document, a source file or an
  in-game observation.
- Keep what was observed apart from what was inferred.
- No code addresses in prose: they move with every build. Point to the source
  that verifies them at load time.
- Mark anything version-specific with the build it was measured on.
- Keep `SKILL.md` short; new material goes in `references/`.
- No em dashes.
- The plugin deliberately has no `version`, so users receive every commit. Run
  `claude plugin validate .` before pushing; one warning about the missing
  version is expected.

## Sources and credits

- **TesmioLoader** and its documentation by MaxLegend (GPL-3.0). Most of the
  engine internals, the reverse-engineering method and the pitfalls are
  summarised from its `docs/`; read them for the evidence and the full story.
- **Republic Mod Loader** by Ultimate-Universe (GPL-3.0): loader architecture
  and plugin API.
- **railspeed** by SosseTurner (GPL-3.0): the infrastructure speed-limit lookup
  and its signatures.
- **RailPhysics** by TheRealMeowMeow00 (GPL-3.0): rail internals.
- **TLMM** by TheRealMeowMeow00 (GPL-3.0): Workshop packaging for TesmioLoader
  plugins.
- Token counts and item layouts were measured on the base game and on
  subscribed Workshop items in September 2026.
- Written with AI coding agents and checked against the sources above.

## License

CC BY 4.0, see `LICENSE`. Copy it into your projects freely; keep the
attribution line at the top of `SKILL.md`.
