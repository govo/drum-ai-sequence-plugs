---
name: drumai-preset
description: Create, edit, or analyze Drum AI drum patterns and return both a direct Drum AI app deep link and the exact PRESET web URL produced by the Drum AI PRESET MCP service. Use for drum beats, grooves, fills, styles, and PRESET rework requests.
metadata:
  version: "1.3"
compatibility: Requires the Drum AI PRESET MCP service to be reachable.
---

# Drum AI PRESET

Use the Drum AI PRESET MCP service to turn a natural-language drum request into an editable Drum AI sequence.

## Core delivery rule

The final deliverable has TWO links, and `render_preset` returns both of them already assembled:

1. `deeplink` — the direct Drum AI app link, of the form `drumai://import?p=<value>`
2. `url` — the web preview/import link, of the form `https://c1c1.online/drumai_mcp/p?p=<value>`

Both carry the same payload and are built server-side from one encoded payload. Copy each field out whole and hand them to the user exactly as received.

Do not modify, decode, re-encode, or truncate either value. In particular, do not build the app link yourself by copying the `p` parameter out of `url` and prepending `drumai://import?p=` — that is hand-transcription of a long base64url string, and a single dropped character makes the link unusable. The `deeplink` field already is the app link.

If a returned URL carries additional query parameters, keep the whole field unchanged as the web link.

The configured MCP endpoint is:

`https://c1c1.online/drumai_mcp/mcp`

Do not substitute another endpoint unless the MCP configuration has been explicitly changed.

## Standard creation flow

For a new sequence, follow this order:

1. Call `list_kits` and select an appropriate kit for the requested style.
2. Call `get_kit` for the selected kit so voice indices and roles are known.
3. Call `list_grid_options` for the requested time signature and grid density.
4. Call `create_draft` with name, BPM, time signature, `cellsPerQuarter`, bars, groove, humanize, and `tags`. Pass `tags` on this call and always give the pattern at least one. Tags are the style labels the Drum AI app filters PRESETs by, so an unlabeled PRESET is hard for the user to find again. Pick from this fixed vocabulary, using as many as genuinely fit: `rock`, `punk`, `pop`, `funk`, `hiphop`, `trap`, `house`, `techno`, `lofi`, `jazz`, `latin`, `rnb`, `reggae`, `country`, `blues`, `metal`, `soul`, `disco`, `edm`, `world`, `afrobeat`. If the style only becomes clear after the pattern is written, set the tags with `update_draft` (`tags`) instead; passing an empty array there clears them. The tags travel inside the exported PRESET, so whoever imports the link gets them too.
5. Call `set_voice_grid` to write the drum pattern. Prefer writing several voices in one call when supported.
6. Add per-hit velocity, ratchet, flam, or voice-mixer detail. Velocity is the main lever for dynamics and the app fully supports it — every hit can carry its own strength from `0` to `1` (`velocities` on `set_voice_grid`, a sparse map keyed by 0-based step index; omitted steps are `1.0`). Do not ship a pattern with every hit at the same strength.
7. For multi-bar patterns, make meaningful bar-to-bar variation rather than blindly repeating one bar.
8. Call `validate_draft`. Never return an empty or invalid pattern.
9. Call `render_preset`.
10. Read the `url` and `deeplink` fields from `render_preset`.
11. Return both links as Markdown links, clearly labeled, with the direct app link (`deeplink`) first. The address must be the Markdown link destination, not code-formatted text.

## Pattern rules

- `triggers` is an absolute grid string across all bars; use it when bars need different patterns.
- `hits` describes positions within a bar and repeats those positions across every bar.
- `cellsPerQuarter` must be one of `2`, `3`, `4`, `6`, or `8`; consult `list_grid_options` instead of deriving positions yourself.
- Use `3` or `6` for triplet grids.
- Use `8` when the requested rhythm genuinely needs 32nd-note resolution, such as dense double-bass figures.
- Velocity step indices are 0-based; `hits.beat` is 1-based.
- Velocity is per step, not per individual hit: the extra hits a ratchet or flam adds at a step share that step's velocity. Accent the backbeat, keep the ghost notes around `0.35-0.6`, alternate strong/weak on the hats.
- Apply fill modes before building velocity dynamics because a fill can replace a voice row and reset velocities.
- Write variation index `0` only. Do not rely on accent or chain because the engine does not audibly use them.
- Keep the kick/snare skeleton clear and leave musical space; do not fill every voice unnecessarily.

## Built-in helpers

Use these when appropriate:

- `apply_style_template`: useful when starting from a style skeleton.
- `apply_fill_mode`: useful for fills, while remembering that it can replace the whole row.
- `list_reference_presets` + `get_reference_preset`: useful for studying the app's reference grooves.
- `parse_preset`: use when the user supplies an existing PRESET/share text or link and wants it edited.
- `render_wireframe`: use for read-only inspection of an existing pattern.

## Delivery format

After `render_preset` succeeds, present the result in this exact Markdown shape, replacing only the two placeholders with the unmodified returned fields:

- [Open in Drum AI](<deeplink>)
- [View web preview](<url>)

Use standard Markdown link syntax so both addresses render as clickable links in ChatGPT. Do not put either address in backticks, a fenced code block, or plain-text-only output. Keep angle brackets only as part of the Markdown destination syntax; replace `<deeplink>` and `<url>` with the exact returned values, without modifying their query strings.

Both must be the exact field values returned by `render_preset` — the app link comes from `deeplink`, the web link from `url`. Neither one is constructed by you.

Do not expose or rewrite the long PRESET payload separately.

If the returned URL has a normal public hostname, it can be handed to the user as-is. If the MCP returns a loopback/localhost URL, report that it is only reachable from the machine hosting the service rather than pretending it is a public web link. The direct app link still requires a valid `p` parameter.

If a previously returned link is reported broken, do not edit the old URL. Re-render the same draft with `render_preset` and hand over the fresh `deeplink` and `url` fields.
