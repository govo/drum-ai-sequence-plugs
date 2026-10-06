---
name: drumai-preset
description: Create, edit, or analyze Drum AI drum patterns and return both a direct Drum AI app deep link and the exact PRESET web URL produced by the Drum AI PRESET MCP service. Use for drum beats, grooves, fills, styles, and PRESET rework requests.
metadata:
  version: "1.1"
compatibility: Requires the Drum AI PRESET MCP service to be reachable.
---

# Drum AI PRESET

Use the Drum AI PRESET MCP service to turn a natural-language drum request into an editable Drum AI sequence.

## Core delivery rule

The final deliverable has TWO links:

1. A direct Drum AI app deep link in this exact form:
   `drumai://import?p=<value>`
2. The original web preview/import URL returned by `render_preset`, unchanged.

The MCP service returns a URL such as:

`https://c1c1.online/drumai_mcp/p?p=xxxxx`

Extract the value of the `p` query parameter and use it to construct the app deep link:

`drumai://import?p=xxxxx`

Do not modify, decode, re-encode, truncate, or otherwise change the `p` parameter value. The web URL must also be returned exactly as received from `render_preset`.

If `render_preset` returns a URL with additional query parameters, preserve the `p` parameter value exactly and keep the entire original URL unchanged as the web link. Do not invent a deep link if the required `p` parameter is absent.

The configured MCP endpoint is:

`https://c1c1.online/drumai_mcp/mcp`

Do not substitute another endpoint unless the MCP configuration has been explicitly changed.

## Standard creation flow

For a new sequence, follow this order:

1. Call `list_kits` and select an appropriate kit for the requested style.
2. Call `get_kit` for the selected kit so voice indices and roles are known.
3. Call `list_grid_options` for the requested time signature and grid density.
4. Call `create_draft` with name, BPM, time signature, `cellsPerQuarter`, bars, groove, and humanize as appropriate.
5. Call `set_voice_grid` to write the drum pattern. Prefer writing several voices in one call when supported.
6. Add velocity, ratchet, flam, or voice-mixer details when they materially improve the requested groove.
7. For multi-bar patterns, make meaningful bar-to-bar variation rather than blindly repeating one bar.
8. Call `validate_draft`. Never return an empty or invalid pattern.
9. Call `render_preset`.
10. Read the `url` field from `render_preset`.
11. Extract the `p` query parameter from that URL without changing its value.
12. Construct `drumai://import?p=<same-value>`.
13. Return both links, clearly labeled, with the direct app link first.

## Pattern rules

- `triggers` is an absolute grid string across all bars; use it when bars need different patterns.
- `hits` describes positions within a bar and repeats those positions across every bar.
- `cellsPerQuarter` must be one of `2`, `3`, `4`, `6`, or `8`; consult `list_grid_options` instead of deriving positions yourself.
- Use `3` or `6` for triplet grids.
- Use `8` when the requested rhythm genuinely needs 32nd-note resolution, such as dense double-bass figures.
- Velocity step indices are 0-based; `hits.beat` is 1-based.
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

After `render_preset` succeeds, present the result in this order:

**Open in Drum AI**
`drumai://import?p=<value>`

**View web preview**
`https://c1c1.online/drumai_mcp/p?p=<value>`

The web preview URL must be the exact URL returned by `render_preset`. The app link is the only URL that may be constructed, and it must be constructed solely from the returned `p` query parameter.

Do not expose or rewrite the long PRESET payload separately.

If the returned URL has a normal public hostname, it can be handed to the user as-is. If the MCP returns a loopback/localhost URL, report that it is only reachable from the machine hosting the service rather than pretending it is a public web link. The direct app link still requires a valid `p` parameter.

If a previously returned link is reported broken, do not edit the old URL. Re-render the same draft with `render_preset` and generate fresh links from the new returned URL.
