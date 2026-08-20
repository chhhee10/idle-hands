# Previous build — what was made, how it worked, why we're starting over

Record of the first attempt (2026-08-15 → 2026-08-16), written before the repo was
cleared. Self-contained: everything below that mattered is restated here, not linked.
The original specs (`game-prd.md`, `PRD-V2.md`) are kept locally alongside this file.

---

## 1. Timeline

| When | What |
|---|---|
| Aug 15 | v1 built from `game-prd.md`: Go server + Postgres, Next.js 15 + Phaser 3 client, npm hook, Docker/Caddy. Dice-roll instant raids on a fixed 4×4 plot grid. Dusk-castle pixel art. Adversarial review → 8 confirmed fixes. |
| Aug 15 | Owner: "I like the design but make it represent India, retro pastel." Full re-theme to a golden-hour Indian bazaar palette and haveli/chhatri building silhouettes. |
| Aug 16 | Owner: "I do not like this game." Wanted Clash-of-Clans-style raids (mouse troop deployment), free base building, browsing/searching/liking other bases, income from token consumption. v2 pivot: 12×12 free-placement yard, 90-second server-authoritative live battles over SSE, three troop types, likes + player search, transcript-based token income. |
| Aug 16 | v2 demoed in the browser: three real battles against a walled base (0% timeout, 9% loss to towers, 48% win via a breached wall; keep stormed). Pushed to `github.com/chhhee10/idle-hands`. |
| Aug 16 | Owner verdict: still not the game they want. Reset to total scratch. |

## 2. What the owner liked / didn't

**Liked**
- The bazaar visual direction: warm peach-to-rose sky, cream havelis with teal chhatri domes, striped vermillion awnings, marigold lamp glow, maroon 9-slice panels. The share-card vignette (banded sun, palm, domed skyline) was the best single frame.
- The idea of agents earning for you while you work.

**Didn't like**
- v1: instant dice-roll raids with no agency ("deterministic and instant" per the PRD) — nothing to *do*.
- v1: the tiny fixed 4×4 plot grid — no sense of building a world.
- v1: no way to see, browse, search, or react to other people's bases.
- v2 (after seeing it): still not it. The battle worked mechanically but the overall game state didn't land. Unstated specifics — treat the whole loop as unproven rather than any one feature as the culprit.

**Explicitly requested at various points**: troop deployment with the mouse, free-world building, seeing other bases "on the server", searching for people in world mode, liking bases, income from token consumption read from transcripts, a retro-pastel Indian look, a reference to Steam's *Protect the Pit* (tower-defence/strategy/build-your-world).

## 3. Architecture that was built (and worked)

- **One Go binary** (`net/http` + pgx), Postgres 16 the only datastore, migrations embedded and applied at boot under an advisory lock. No Redis, no queues, no websockets.
- **Lazy accrual, no tick loop**: every money-touching endpoint called one central `settleUserTx` (lock user row → lazy season reset → lazy building-state normalization → read best ok-forge → pay minutes in (last_accrued, now-1] → advance marker). A town nobody opens costs nothing.
- **SSE** for world events (raids, shield expiry, season end) and, in v2, per-battle event streams. In-process pub/sub broker keyed by (world, user); 25 s pings.
- **GitHub OAuth** (HMAC-signed state), DB-backed cookie sessions (128-bit ids, sliding 30-day). A `GAME_DEV_LOGIN=1` backdoor (`POST /auth/dev {login}`) for local play without OAuth — loud boot warning.
- **Next.js 15 app router + Phaser 3** in a single client component; integer-only pixel scaling; 9-slice `border-image` panels from generated PNGs; plain CSS custom properties; dev rewrites proxying `/v1` and `/auth` to `:8080`; standalone output in Docker.
- **npm hook** with zero dependencies: Claude Code `PostToolUse` hook entry → `beat.js` drains stdin unparsed, aggregates a per-minute count, flushes ≤1/60 s, spools offline (5 MB cap, drop oldest), mkdir-lock against concurrent beats, at-most-once delivery. v2 added content-blind tailing of `~/.claude/projects/**/*.jsonl` extracting only numeric `usage.*` fields.
- **Procedural pixel art**: a Go generator (stdlib only) with a painting kit (iso boxes, dither, outlines) renders every sprite, packs a Phaser JSON-hash atlas, emits 9-slice UI PNGs, a share-card vignette, a favicon, and a MANIFEST. Atlas frame names were a hard contract with the client, so re-themes changed pixels only.
- **Deploy**: docker-compose (postgres, server, web, caddy), build context = repo root, Caddy routes `/v1/* /auth/* /healthz` → server, else → web; `{$GAME_DOMAIN:http://localhost}` for auto-TLS.

Repo layout: `/cmd/server`, `/internal/game` (pure rules, 96% coverage), `/internal/game/battle` (v2 sim), `/internal/store`, `/internal/http`, `/migrations`, `/web`, `/hook`, `/assets/gen` (own go.mod), `/deploy`.

## 4. Game logic as implemented

### 4.1 Economy
- Credits are **global per user**; buildings are **per town per world**.
- Integer precision via **shards = 1/600 credit** (`users.credit_frac` carries the remainder).
- **Session income per active minute** (a minute with `tool_calls > 0`): `270 + 81·forge_level` shards = `0.45·(1 + 0.30·forge)` credits. Forge 0..5 → 27.0 / 35.1 / 43.2 / 51.3 / 59.4 / 67.5 credits per hour.
- **Token bonus (v2)**: `base · min(tokens_in_minute, 30000) / 30000` extra shards — up to 2× at ≥30k tokens/min. Integer division last.
- **Floor**: 80 shards/min (8 credits/hr) for the first 360 minutes of each UTC day (minute mod 1440 < 360), additive, never token-scaled. 48 credits/day for everyone.
- **Farm cap**: `session_minutes (user_id, minute_ts)` primary key — ten parallel sessions in one minute = one row. Ingest upserts additively; rejects minutes >10 min future or >24 h past; ≤120 beats/POST; token-bucket 1 req/10 s burst 6.
- **Settle window**: `(last_accrued_minute, now_minute − 1]` — the open minute is never paid. New users start with `last_accrued = now − 1` (back-submitted history before account creation pays nothing).
- **Forge while damaged/rubble** counts as level 0 (base rate continues). Effective forge level is read *before* normalizing lapsed damage so a damaged window is never retroactively paid at full rate.
- **Liveness**: an active minute at `now` or `now − 1` (the PRD's 90-second rule).

### 4.2 Buildings and costs
- Kinds: keep (account level), forge (income), granary (loot cap), watchtower (defence), wall (defence), barracks (offence).
- `cost(kind, L) = round10(base · 1.65^(L−1))`, half-up; base = keep 400, forge 260, granary 180, watchtower 200, wall 140, barracks 240. Keep L1=400, L2=660, L3=1090, L4=1800, L5=2960, L6=4890; forge L2=430, L3=710.
- Repair = `round10(0.40 · cost(kind, current_level))`. Cumulative value Σ cost(1..L) = standings score basis.
- Caps: keep ≤ 7, forge ≤ 5, barracks ≤ 5, everything else ≤ keep level. Instant upgrades, no timers.
- **v1 plots**: 4×4, `unlocked = 4 + (keep−1)·2`; canonical unlock order `(1,1)(2,1)(1,2)(2,2) (1,0)(2,0) (0,1)(3,1) (0,2)(3,2) (1,3)(2,3) (0,0)(3,0) (0,3)(3,3)`, keep at (1,1); one each of forge/granary/barracks, many towers/walls.
- **v2 yard**: 12×12 cells, outer ring = deploy ground (unbuildable). Keep 2×2 at (5,5) by default, all else 1×1. Counts by keep level 1..7: towers = keep level; wall segments 8/14/22/30/40/50/60; one forge/granary/barracks. Walls bought per segment at `cost(wall,1)`, upgraded as a **group** for `round10(cost(wall,L)·segments/10)`; `towns.wall_level` authoritative. `POST layout` takes the whole placement list, validates bounds/ring/overlap atomically; rejected while the town is a live-battle defender.

### 4.3 Damage lifecycle
- Sacked/destroyed → `damaged`, `state_until = now + 6h`, auto-repairs lazily; instant repair for 40%.
- Sacked again within 24 h of `last_sacked_at` → `rubble` for 14 h, cannot be bought out, then returns to ok at the same level.
- Damaged/rubble buildings contribute nothing (no income bonus, no defence, no loot cap).

### 4.4 Raids v1 (dice, removed in v2)
- Preconditions in order: attacker idle → 30-min cooldown (`towns.last_raid_at`) → target unshielded → same world → not self.
- Seed `(attacker_id·1_000_003) ⊕ (defender_id·998_244_353) ⊕ unix_minute`, splitmix64. `waves = barracks_level` (min 1), `wave_power = 10 + 6·barracks + rng(0..8)` rolled once; towers absorb one wave per level; `wall_soak = 4·Σwall_levels`; target priority forge > granary > barracks > watchtower > wall, keep immune.
- Loot `= round(min(def_credits·0.20, 150·(1+granary)) · sacked/waves · gap_mult)`, `gap_mult = 1` if attacker keep ≤ defender keep else `max(0.25, 1 − 0.25·gap)`.
- Shield 8 h iff ≥1 building sacked; raiding drops the attacker's own shield.

### 4.5 Battles v2 (live)
- `POST raid` → freezes the defender's yard into a snapshot, inserts a `battles` row, drops attacker shield, starts cooldown, registers an in-memory session. One global 10 Hz ticker steps all live battles. `GET /v1/battles/:id` (rejoin) + SSE events 4 batches/s + `POST deploy {troop, x, y}` on ring cells only. One live battle per attacker and per defender. Server restart voids live battles with no effects.
- Fixed 100 ms tick, max 900 ticks (90 s). Ends on time, 100% destruction, or troops dead + capacity < cheapest troop. Integer units on the wire: milli-cells, deci-hp.
- Capacity `10 + 6·barracks_level`. Troops: **raider** 2 pts, 60 hp, 12 dps, 2.0 cells/s, nearest building; **breaker** 5 pts, 200 hp, 20 dps (×4 vs walls), 1.0, nearest wall/tower; **looter** 3 pts, 35 hp, 8 dps, 3.0, nearest forge/granary. Fallback to nearest non-wall building when preferred targets are gone.
- Watchtower L: range 3.5 cells, `10 + 4L` dps, retargets nearest troop each tick. HP: walls `80·L`, keep `300 + 100·L`, others `100 + 60·L`.
- BFS pathing on the cell grid; walls are destructible blockers; unreachable target → attack the nearest blocking wall.
- Destruction % = destroyed non-wall hp / total non-wall hp. Loot = `round(min(def_credits·0.20, granary_cap) · destruction% · gap_mult)` + 10% per looter-kill of a resource building (capped at the pool). Win iff destruction ≥ 50% or keep fell; shield iff ≥ 30%.
- Observed balance: two L2 towers shredded a 9-troop spread deployment (9%); a concentrated 27-point assault through a pre-breached wall face stormed the keep at 48%. Breaching walls in one raid and exploiting it in the next was emergent and fun.

### 4.6 Worlds, seasons, social
- Worlds ≤ 40 members; team worlds (invite code, org-mate suggestions from GitHub `read:org`) and open worlds (newest with space, seasonal, closed at rollover).
- **Seasons are globally synchronized** 14-day windows from `GAME_SEASON_EPOCH` (2026-08-10T00:00Z). Lazy rollover on touch: users reset credits; worlds snapshot `season_results`, reset towns to keep-only L1, close public worlds.
- Standings score = cumulative town value + credits looted this season in this world; W/L shown; never session minutes or tokens.
- Likes: one per (liker, likee, world, season). Player search across own + public worlds, login/avatar/worlds only. Public town pages `/t/[world]/[login]` OG-tagged with a 1200×630 ImageResponse card; season summary cards from `season_results`.
- Moderation seam: `reports` table + `POST report` on public worlds only; team worlds 404 it.

### 4.7 Privacy rules that shaped everything
- Telemetry opt-in (minting a token). Only minute buckets and counts ever leave the machine; the hook never parses the Claude Code payload; transcript tailing reads only numeric usage fields. No table holds content, paths, or repo names.
- Other players' town payloads never include credits, income, or activity. Standings never sortable by activity.
- One-click revoke deletes all tokens and all `session_minutes` rows.
- A permanent line under the session banner: "This is a game, not a productivity metric…"

## 5. API surface (final shape)

`GET /v1/me` · `GET /v1/worlds/:slug` (standings, invite code for members) · `GET /v1/worlds/:slug/town` (own: credits, income, deploy_capacity, raid_gate, server-priced `upgrades[]`) · `GET /v1/worlds/:slug/towns/:login` (no credits) · `POST upgrade {kind, building_id?, build_new?}` · `POST repair {building_id}` · `POST layout {placements[]}` · `POST raid {target_login}` → battle session · `GET /v1/battles/:id` · `GET /v1/battles/:id/events` (SSE) · `POST /v1/battles/:id/deploy` · `GET raids` (last 20, with direction) · `GET events` (SSE) · `GET suggested` · `POST/DELETE like` · `GET /v1/search/players?q=` · `POST /v1/worlds`, `POST /v1/worlds/join` · `POST /v1/ingest` (bearer) · `POST/GET/DELETE /v1/tokens`, `POST /v1/telemetry/revoke` · `POST report` · `GET /v1/public/towns/:slug/:login`, `GET /v1/public/seasons/:slug/:n` · `/auth/github`, `/auth/github/callback`, `/auth/logout`, `/auth/dev` (dev only) · `/healthz`.

Errors `{error, code}`; codes included `attacker_live, raid_cooldown, target_shielded, not_in_world, self_raid, battle_in_progress, insufficient_credits, count_cap, keep_level_cap, max_level, duplicate_kind, building_damaged, out_of_bounds, deploy_ring, overlap, not_deploy_ground, not_attacker, bad_token, rate_limited`.

The client never recomputed economy math — every cost, gate, and reason came from the server.

## 6. Art direction (final)

Palette: sky `#f2a97e → #d97e63`, plaza sand `#dca55f` / `#cb924e`, terracotta edge `#a5623a`, cream `#f7e8c6`, vermillion `#d84a2b`, marigold `#f2b736`, leaf `#4e9a5f`, teal `#23695f`, maroon `#5c2330`, ink `#38221a`, hostile ember `#a8261a`, flames `#e8543f → #f2b736`, outline `#1f130e`. Fonts: Silkscreen (display), Nunito (body); monospace only inside the raid log.

Buildings (3 tiers each, 88 px wide on an 88×44 iso footprint): keep = haveli tower with chhatri dome and pennant; forge = workshop with striped vermillion awning and furnace (lit variants); granary = round clay matka store with sacks; watchtower = slender chhatri tower with brazier; wall = sandstone with jali band; barracks = akhara hall. Effects: fire_0..5, smoke_0..5, rubble, glow, shield_pale, select_ring, locked_plot, deploy_tile(_bad), crack_1/2, explode_0..3, hit_puff_0..2, muzzle_0..1, bolt. Troops 24×32, SE-facing, mirrored for SW: raider (turban, lathi, vermillion sash), breaker (pehlwan with clay gada, teal langot), looter (marigold headwrap, jute sack). UI: 48×48 9-slice panels (16 px insets) in cream/maroon with painted wood + teal border and brass pins; 30×30 buttons (10 px insets) with up/down/disabled states.

Banned throughout: monospace UI font, lowercase-as-style, `[bracket]` buttons, hairline boxes, near-black backgrounds, the FailproofAI pink, emoji.

## 7. Known issues at reset

- v2 server and client shipped agent-tested (including a curl-driven full battle) but never received the adversarial review pass v1 did.
- Troop bar displayed deci-hp as hp ("6 hp" for a 60-hp raider).
- The battle's 90-second clock runs in real time from raid confirmation; there is no "deploying" pre-phase, so hesitation is punished and latency between actions matters.
- The world page's layout shifted on live banner refetches, moving buttons under the cursor.
- The neighbour strip's raid-gate reason could lag the banner by one refetch.
- Hook `tokens` field: v1 always 0; v2 tails transcripts — correct, but the field's meaning changed between versions.
- No defender "you are under attack right now" notification; defenders learned of raids only when the battle committed.

## 8. Lessons for the rewrite

**Design**
- "Deterministic and instant" raids had no agency and were the core complaint. Interactive combat fixed the *moment* but not the *loop*; the owner still wasn't happy, so the problem is probably the loop (what do I do when I open the tab? why come back?) rather than the combat system. Design the daily loop first, combat second.
- Free placement only matters if layout decisions are visible and consequential; a 12×12 yard with five buildings felt empty. Either give many cheap things to place (decor, paths, gardens) or make the map smaller and denser.
- Browsing other bases, search, and likes were asked for before they were built — social discovery should be a first-class screen, not a side panel.
- A real-time 90-second window inside an asynchronous game is an awkward hybrid; consider either a proper "plan then watch" (deploy phase, then replay) or embrace short synchronous sessions.

**Technical (keep)**
- Lazy accrual with a single settle function, integer shard math, and the `(user, minute)` primary-key farm cap all worked flawlessly and were trivially verifiable.
- Pure-Go rules package with exhaustive tests caught everything before integration; the battle engine's byte-identical replay test made server authority cheap.
- Server-sent events were enough for both world notifications and a 10 Hz battle stream.
- Procedural art from a Go generator with a frozen frame-name contract made two full re-themes painless and kept licensing trivial (everything original).
- Dev-login backdoor + docker-compose override made end-to-end browser demos fast.
- Writing `DECISIONS.md` (ambiguity resolutions) and `API.md` (exact wire shapes) *before* fanning out parallel builders prevented drift between server and client.

**Technical (change)**
- Decide the yard/grid model before building the renderer; the 4×4 → 12×12 change rewrote the scene, the store, the migrations, and the public pages.
- If battles are live, bake latency tolerance into the client (queue deploys, generous clocks, an explicit pre-battle deploy phase).
- Review every layer before demoing; the unreviewed v2 had display bugs a reviewer would have caught in minutes.

## 9. Process notes

Built almost entirely by parallel coding agents orchestrated in workflows: disjoint-path ownership per agent, machine-read gate reports (tests actually run, honestly reported), an adversarial review stage (finders → skeptic verifiers) that surfaced 23 findings with 8 confirmed, and end-to-end browser verification with real Postgres, real ingest, and real battles. Two agents died on a usage limit mid-run and were resumed from the workflow cache without redoing finished work.
