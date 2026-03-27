# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Fetch data for a GitHub org (reads GITHUB_TOKEN/.env)
uv run fetch.py <org>
uv run fetch.py <org> --skip-forks --skip-archived --limit 10

# Serve the visualization locally (open http://localhost:8000)
python3 -m http.server 8000 --directory docs
```

No build step, no tests — `docs/index.html` is a self-contained single-file app.

## Architecture

Two files do all the work:

- **`fetch.py`** — Python script that hits the GitHub API and writes `docs/data.json`. Reads package manifests (`package.json`, `requirements.txt`, `pyproject.toml`, `go.mod`, `Cargo.toml`, `Gemfile`, etc.) to extract dependency names, then cross-references them against org repo names to build internal dependency edges.

- **`docs/index.html`** — ~2500-line single-file app (HTML + CSS + JS). Uses D3.js for force simulation, marked.js for README rendering. No bundler, no build step — everything is inline.

### Front-end layout

```
#app
  #sidebar-left   — Settings (filters, force layout controls, global section toggles)
  #graph-container
    #edges-svg    — SVG layer for edges (D3 paths + arrowhead markers)
    #nodes-layer  — DOM layer for repo/contributor node tiles (CSS-transformed)
  #sidebar-right  — Stats (LoC, aggregate commit graph, top deps/contributors)
```

### Two-tab graph view

A tab bar at the top of `#graph-container` switches between:

- **Repos tab** — dependency graph: repo nodes, directed edges (arrows) from dependents to dependencies.
- **Contributors tab** — co-contribution graph: contributor avatar nodes, undirected weighted edges (thickness = `log(shared_repos)`).

`switchTab(tab)` swaps sidebar section visibility and re-initializes the active simulation. The `contribInitialized` flag avoids rebuilding the contributor layout on every tab switch.

### Key globals

| Variable | Purpose |
|---|---|
| `activeTab` | `'repos'` or `'contributors'` |
| `nodes/links/simulation/edgeEls` | Repo graph state |
| `contribNodes/contribLinks/contribSimulation/contribEdgeEls` | Contributor graph state |
| `vpTransform` | Shared pan/zoom state (both tabs share viewport) |

### Pan/zoom

Mouse drag on `#graph-container` updates `vpTransform` and applies a CSS `transform` to both `#edges-svg` and `#nodes-layer`. Nodes are positioned with `position: absolute` and `translate()` set directly on their style.

### `data.json` schema

```json
{
  "org": "string",
  "generated_at": "ISO8601",
  "total_repos": 42,
  "repos": [{
    "id": "repo-name",
    "name": "repo-name",
    "languages": { "Python": 12345 },
    "contributors": [{ "login": "...", "avatar_url": "...", "commit_count": 10, "pr_count": 2 }],
    "commit_frequency": [0, 3, ...],   // 52 weekly totals
    "readme": "string|null",
    "dependencies": { "external": ["pkg-name"], "internal": ["other-repo"] }
  }]
}
```

## CI / GitHub Pages

`.github/workflows/update-viz.yml` runs `fetch.py` and commits `docs/data.json` on push to main and every Monday at 03:00 UTC. Publish by enabling GitHub Pages from `main` / `/docs`.
