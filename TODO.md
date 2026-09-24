# TODO

## Run on the Python 3.14 plugin host (upstream issue #208)

Assessed 2026-09-24. Not started.

Since Sublime Text 4213, Python 3.8 was replaced by 3.14 and the legacy 3.3 host is disabled by default (and slated for removal). Mediawiker ships no `.python-version`, so it runs on the 3.3 host, which currently works only because `Preferences.sublime-settings` has `"disable_plugin_host_3.3": false`. See <https://github.com/tosher/Mediawiker/issues/208>.

Estimate: an afternoon (2--4 hours), mostly dependency swaps and restart-and-read-the-console iteration rather than code changes. Deferring the jinja2 imports alone (the issue's "smaller change") is **not** sufficient, because `requests` also fails at import.

### Blockers, in the order they will likely surface

- [ ] **jinja2 2.10.1** (PC3 dependency `python-jinja2`) does `from collections import Mapping`, removed in Python 3.10. Imported at module scope in `mediawiker.py:7` and `mwcommands/mw_preview_page.py:7`. Only used by `render_page_template()` (`new_page_template_path` setting) and HTML preview.
- [ ] **requests 2.15.1** (PC3 dependency `requests`) has the same dead import in `cookies.py`, `sessions.py`, `models.py`, `structures.py`, `utils.py`, and vendored `urllib3/_collections.py`. Imported at module scope in `mwcommands/mw_utils.py:13`, so this blocks the whole plugin.
    - Fix for both: move `dependencies.json` to PC4 libraries (likely `requests`, `Jinja2`, `MarkupSafe`, `oauthlib`, `requests-oauthlib`; verify exact names in the PC4 channel) and add a `.python-version` containing `3.8` (3.8-targeted packages run on 3.14).
- [ ] **Bundled `lib/mwclient`** vendors an old `six.py` whose meta-path importer uses `find_module`/`load_module`, which Python 3.12 no longer calls, so `six.moves` imports may break. Fix: drop in a current `six` (1.16+ has `find_spec`), or upgrade to a modern `mwclient` (which dropped `six`; may need small API changes). This is the main unknown.
- [ ] **Bundled `lib/keyring.zip`** is imported at module scope via `lib/browser_cookie3/__init__.py:33` (needed for Firefox cookie auth). Uses its own vendored `entrypoints` and a ctypes macOS backend. Probably fine; unverified.
- [ ] **`lib/Crypto.osx.x64`** has x86_64 `.so` files built for old Python; they won't load on 3.14/arm64. The import is in a `try` (`browser_cookie3/__init__.py:47`), so only Chrome cookie decryption breaks. Firefox cookies (current setup) are unaffected. Low priority.

### How to work on it

1. Start a separate jj change so `master` stays usable.
2. Set `"disable_plugin_host_3.3": true`, restart Sublime, and read the traceback in the console (`ctrl+backtick`). Fix, restart, repeat.
3. To get a working Mediawiker back mid-way, set the flag back to `false` and restart.
4. When done, consider a PR upstream referencing #208 (upstream inactive since 2024-08), and remove the workaround from `Preferences.sublime-settings`.

## Community adoption (after the 3.14 fix works)

Upstream appears abandoned. The [SublimeText](https://github.com/SublimeText) GitHub org ("Collection of Sublime Text packages maintained by the community") adopts such packages. The process below is convention from memory, not written policy; confirm on their [Discord](https://discord.gg/D43Pecu) first.

- [ ] Ask tosher (e.g., on #208) to transfer the repo to the SublimeText org or add maintainers. A transfer keeps issues, stars, and redirects.
- [ ] If no response after a reasonable wait, ask on the Discord about forking into the org.
- [ ] If forked rather than transferred, PR `wbond/package_control_channel` to change `"details": "https://github.com/tosher/Mediawiker"` in `repository/m.json` (line 1351 as of 2026-09-24) to the new repo.
- [ ] Add the org repo as a remote in this clone (`jj git remote add community ...`).
