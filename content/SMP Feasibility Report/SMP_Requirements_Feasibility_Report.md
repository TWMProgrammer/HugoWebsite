# SMP Requirements Feasibility & Implementation Checklist

Source inputs:
- [Requirements.md](file:///v:/crafty/servers/04c9743c-ab3a-444e-95ff-21633e93f4d5/Documents/Requirements.md)
- [SMP_Production_Report.md](file:///v:/crafty/servers/04c9743c-ab3a-444e-95ff-21633e93f4d5/Documents/SMP_Production_Report.md)

This report checks whether the requirements are achievable with the current installed plugins/configs, and provides a concrete implementation checklist aligned with the production-readiness direction (performance, security, stability, economy integrity).

Legend:
- DOABLE: achievable via configuration + existing plugins.
- PARTIAL: achievable, but requires non-trivial custom wiring (Skript/zMenu) and/or careful policy decisions.
- NEEDS ADDITIONAL: requires adding a plugin or custom development beyond configuration.

## 1) Current Snapshot (What You Have Right Now)

### 1.1 Server Core
- Version target: 1.21.x+ (Paper config present).
- Current auth mode: `online-mode=false` in [server.properties](file:///v:/crafty/servers/04c9743c-ab3a-444e-95ff-21633e93f4d5/server.properties#L44-L45) (production blocker per production report).
- Performance-relevant current values:
  - `view-distance=12`, `simulation-distance=12` in [server.properties](file:///v:/crafty/servers/04c9743c-ab3a-444e-95ff-21633e93f4d5/server.properties#L61-L69)
  - `max-chained-neighbor-updates=1000000` in [server.properties](file:///v:/crafty/servers/04c9743c-ab3a-444e-95ff-21633e93f4d5/server.properties#L38-L38)
  - Paper anti-xray currently disabled in [paper-world-defaults.yml](file:///v:/crafty/servers/04c9743c-ab3a-444e-95ff-21633e93f4d5/config/paper-world-defaults.yml#L17-L52)

### 1.2 Installed Plugin Inventory (JARs)
Core & management:
- EssentialsX (+ Chat/Spawn) + EssentialsDiscord (+ Link)
- LuckPerms (+ LuckPermsGUIPlus)
- Vault, PlaceholderAPI, ProtocolLib, PacketEvents
- Tebex
- ViaVersion

World & performance:
- Paper + Chunky (pregen)
- FastAsyncWorldEdit
- TerraformGenerator (custom worldgen)
- ClearLaggEnhanced

Economy & trade:
- CrazyAuctions (auction house)
- ExcellentCrates (crates + milestones/pity capability)
- VanguardRanks (rank progression)
- AxTrade (safe player trades)
- nightcore (economy/currency engine used by some plugins)
- DailyRewards (login rewards)

Hardcore / PvP systems:
- LifeStealZ (hearts, elimination, revive beacon item)
- PvPManager (combat tagging / protections)
- AntiHealthIndicator, AntiPopup

Protection / moderation:
- CoreProtect
- LibertyBans
- Protect (WorldGuard-style areas) (folder present; config not generated yet)

Death mechanics:
- AxGraves (death graves)

UI & visuals:
- TAB
- DecentHolograms
- ajLeaderboards
- zMenu

## 2) Requirement Feasibility (By Spec Section)

### 2.1 Core Server Model (hardcore-leaning SMP, PvP/raiding, economy-driven)
- Status: DOABLE
- Notes:
  - The current stack supports PvP, combat tagging, and a player-driven market (AH + trading).
  - The main risk is accidental “safety” features (teleports, graves, back-on-death, generous rewards) undermining the hardcore economy loop.

### 2.2 Technical Direction (1.21.x+, custom worldgen, reuse, performance-first)
- 1.21.x+: DOABLE
- Custom world generation: DOABLE
  - TerraformGenerator is installed; you still need to actually assign it to the world(s).
- Performance-first: DOABLE
  - Requires tuning view/simulation distances and lag vectors, plus pregen.

### 2.3 Risk, Death & Item Destruction (primary economic engine)
- Status: PARTIAL
- What works:
  - PvP is enabled; full-loot is vanilla by default.
  - PvPManager is not currently altering drops (`Player Drop Mode: ALWAYS`), which is good for “full loot”.
- What undermines “meaningful loss” today:
  - AxGraves currently keeps 100% XP and makes retrieval highly reliable ([AxGraves config](file:///v:/crafty/servers/04c9743c-ab3a-444e-95ff-21633e93f4d5/plugins/AxGraves/config.yml#L47-L76)).
  - Essentials grants `/back` and `/back on death` to players by default (see `player-commands` in [Essentials config](file:///v:/crafty/servers/04c9743c-ab3a-444e-95ff-21633e93f4d5/plugins/Essentials/config.yml#L256-L330)).

### 2.4 Hearts, Revival & Hardcore Resources (LifeStealZ-aligned)
- Hearts (transfer on kill, lose on death): DOABLE (LifeStealZ is installed and already set to `startHearts: 10`).
- Elimination (temporary ban at “0 hearts”): DOABLE
  - LifeStealZ supports elimination bans (`disablePlayerBanOnElimination: false`) and custom elimination commands.
- Minimum heart floor “5 hearts”: PARTIAL
  - LifeStealZ’s `minHearts` is the elimination threshold ([LifeStealZ config](file:///v:/crafty/servers/04c9743c-ab3a-444e-95ff-21633e93f4d5/plugins/LifeStealZ/config.yml#L29-L43)), so you must choose one of these interpretations:
    - Option A (simplest): set elimination at 5 hearts (treat “floor 5” as the elimination point).
    - Option B (match text literally: floor 5 but eliminate at 0): needs custom logic (likely Skript) to prevent hearts dropping below 5 while still allowing a separate “0 hearts” elimination state.
- Revival recipe and “revive returns with 5 hearts”: DOABLE
  - LifeStealZ has a revive beacon item and craft recipe in [items.yml](file:///v:/crafty/servers/04c9743c-ab3a-444e-95ff-21633e93f4d5/plugins/LifeStealZ/items.yml#L74-L128).
  - Current `reviveHearts` is `1` and must be changed to `5` if you want “revived player returns with 5 hearts”.
- Totems “do not prevent heart loss (recommended)”: PARTIAL
  - If totems are allowed, they typically prevent a “death” event, so heart loss may not trigger.
  - You can either (a) block totems (`preventTotems: true`) or (b) implement custom logic to still apply heart penalties when a totem pops.

### 2.5 World Safety Structure (spawn city + trading posts + limited buffers)
- Status: PARTIAL
- What you have:
  - Protect plugin is present (WorldGuard-like), but no config/data exists yet in `plugins/Protect/` beyond translations, so it likely has not been initialized/used.
- What’s missing:
  - Concrete region definitions and flag policies (no-build, no-pvp, no-explosions, limited buffers).
  - BetterRTP cannot “respect” Protect areas, so spawn safety requires separate RTP configuration (min radius / center away from spawn).

### 2.6 Economy Backbone (money + gems, anchors, sinks)
- Soft currency (Money): DOABLE
  - Vault is installed; Essentials provides balances; CrazyAuctions uses Vault.
- Hard currency (Gems): PARTIAL
  - There is no second “Vault economy” installed, but you can implement Gems as:
    - Item-based currency via nightcore (e.g., a special Gem item sold by Tebex) and use it for crates/menus/black market, or
    - A Skript-managed virtual balance (shown via PlaceholderAPI) with commands/menus charging it.
  - If you require a full second virtual currency with broad plugin support (AH/shop/rankup), that is NEEDS ADDITIONAL.

### 2.7 Server Shop (/shop), Auction House (/ah), Black Market
- Auction House (/ah): DOABLE (CrazyAuctions installed).
- Server Shop (/shop baseline resources only): NEEDS ADDITIONAL (or custom build)
  - There is no dedicated shop plugin in the current JAR list.
  - You can build a controlled shop using zMenu + Skript + Vault, but it’s an implementation project, not “configure-only”.
- Black Market (rotating stock, gems only): PARTIAL
  - Achievable via zMenu + Skript + nightcore item currency (or Skript currency), but needs custom implementation and rotation tooling/policy.

### 2.8 Crates & Gambling (ladder, visible loot tables, pity)
- Status: DOABLE
  - ExcellentCrates is installed and milestones are enabled in [ExcellentCrates config](file:///v:/crafty/servers/04c9743c-ab3a-444e-95ff-21633e93f4d5/plugins/ExcellentCrates/config.yml#L34-L40).
  - You still need to define the actual crate ladder (Vote/Basic/Epic/Ultra) and key pricing/loot, and enable broadcasts for rare wins.

### 2.9 Progression & Ranks (/rankup ladder, donor ranks)
- In-game /rankup: DOABLE (VanguardRanks installed)
  - Requires redesign of the ranks.yml to match your economy sinks instead of default “eco give” rewards.
- Donor ranks: DOABLE
  - LuckPerms + Tebex can grant groups/perks; keep them efficiency/status, not raw power.

### 2.10 Events & Resets (KOTH, LMS, tournaments, seasonal resets)
- Status: NEEDS ADDITIONAL (for full feature set)
  - There are no dedicated event plugins installed for KOTH/LMS/tournaments.
  - You can script small events (basic timed PvP arenas, reward distribution) with Skript, but KOTH/LMS “production grade” typically benefits from purpose-built plugins.
- Resets: DOABLE (operational process)

## 3) Concrete Implementation Checklist (Configurations You Need To Do)

This checklist is written as “apply these settings / build these configs” items. It intentionally focuses on concrete actions rather than design prose.

### 3.1 Production Blockers (Must Fix Before Any Public Test)
- Set authentication policy and make plugins consistent:
  - Set `online-mode=true` in [server.properties](file:///v:/crafty/servers/04c9743c-ab3a-444e-95ff-21633e93f4d5/server.properties#L44-L45) before collecting any real player data.
  - Align LibertyBans auth mode with the server mode (LibertyBans is configured for ONLINE while the server is offline-mode per production report).
- Rotate and remove secrets from disk/configs:
  - Rotate any Discord tokens/webhooks found in EssentialsDiscord and VanguardRanks configs and remove them from shareable configs.
  - Replace the management server secret in `server.properties` if it has been exposed.

### 3.2 Performance Baseline (Stability > Feature Bloat)
- Reduce chunk simulation load:
  - Set `view-distance` to 7–8 and `simulation-distance` to 5–6 in [server.properties](file:///v:/crafty/servers/04c9743c-ab3a-444e-95ff-21633e93f4d5/server.properties#L61-L69).
- Reduce redstone update exploit surface:
  - Lower `max-chained-neighbor-updates` from 1,000,000 in [server.properties](file:///v:/crafty/servers/04c9743c-ab3a-444e-95ff-21633e93f4d5/server.properties#L38-L38).
- Decide ore scarcity policy:
  - If ore scarcity matters economically, enable Paper anti-xray in [paper-world-defaults.yml](file:///v:/crafty/servers/04c9743c-ab3a-444e-95ff-21633e93f4d5/config/paper-world-defaults.yml#L17-L52).
- World pregen before launch:
  - Use Chunky to pre-generate to your intended border, then lock border expectations for the season.
- Resolve scoreboard/nametag ownership:
  - TAB and PvPManager both use scoreboard teams; pick one to own teams to avoid conflicts (see production report note).

### 3.3 Hardcore Loss Policy (Make Death Matter)
- Decide whether graves exist:
  - If you keep graves, tune AxGraves to preserve the “lootable, risky” loop:
    - Reduce `xp-keep-percentage` below `1.0` in [AxGraves config](file:///v:/crafty/servers/04c9743c-ab3a-444e-95ff-21633e93f4d5/plugins/AxGraves/config.yml#L47-L76).
    - Consider disabling `auto-equip-armor` and limiting grave lifetime (if supported in config).
  - If you want pure destruction, disable/remove AxGraves usage and rely on vanilla drops.
- Remove “risk bypass” commands for default players:
  - Remove `/back` and `/back on death` from default permissions (Essentials `player-commands`) if you want deaths to have real consequences.

### 3.4 Hearts & Revival (LifeStealZ)
- Set revival output to 5 hearts:
  - Set `reviveHearts: 5` in [LifeStealZ config](file:///v:/crafty/servers/04c9743c-ab3a-444e-95ff-21633e93f4d5/plugins/LifeStealZ/config.yml#L29-L37).
- Decide the “minimum heart floor” interpretation:
  - If “floor 5” is the elimination threshold, set `minHearts: 5`.
  - If “floor 5 but eliminate at 0” is mandatory, plan a Skript layer to clamp heart loss at 5 and implement a separate elimination state.
- Set elimination to temporary ban (time-based):
  - Configure `eliminationCommands` in LifeStealZ to apply a temporary ban via LibertyBans, instead of a permanent ban flow.
- Revival beacon recipe (match your anchor):
  - Update the revive beacon recipe and/or create a separate “Revival Beacon Core” custom item in [LifeStealZ items.yml](file:///v:/crafty/servers/04c9743c-ab3a-444e-95ff-21633e93f4d5/plugins/LifeStealZ/items.yml#L74-L128) that consumes:
    - 3 Nether Stars
    - 64 Diamond Blocks
    - 1 Heart (LifeStealZ heart item)
    - plus the beacon component(s) you want as the craft spine
- Totem policy:
  - If totems must not bypass heart loss, either:
    - Block them via `preventTotems: true` in [LifeStealZ config](file:///v:/crafty/servers/04c9743c-ab3a-444e-95ff-21633e93f4d5/plugins/LifeStealZ/config.yml#L89-L98), or
    - Implement a “totem pop applies heart loss” handler via Skript.

### 3.5 Safezones (Spawn City, Trading Posts, Buffers)
- Initialize and use Protect for staff-controlled areas:
  - Start the server once to generate Protect config/data (currently only translations exist under `plugins/Protect/`).
  - Define areas for:
    - Spawn City (no build, no damage, no explosions, no stealing)
    - Trading Posts (limited build rules, no PvP, no explosions; allow trading)
    - Onboarding buffers (small radius, no build)
- Configure BetterRTP to avoid hubs:
  - Set an RTP center far from spawn (or raise min radius) in [BetterRTP config](file:///v:/crafty/servers/04c9743c-ab3a-444e-95ff-21633e93f4d5/plugins/BetterRTP/config.yml) so random teleports don’t land inside protected hubs.

### 3.6 Economy Controls (Prevent “Finished” States)
- Remove diamond/emerald injections from DailyRewards:
  - Edit [DailyRewards config](file:///v:/crafty/servers/04c9743c-ab3a-444e-95ff-21633e93f4d5/plugins/DailyRewards/config.yml#L152-L194) to avoid handing out diamonds/emeralds on weekly/monthly timers (replace with low-impact supplies or cosmetics).
- Lock down server-selling:
  - Reduce Essentials sell list in `plugins/Essentials/worth.yml` to baseline items only, with conservative prices, to avoid turning mining/farms into infinite money printers.
- Rebuild /rankup to be sink-first:
  - Replace default VanguardRanks rewards (`eco give`) in [ranks.yml](file:///v:/crafty/servers/04c9743c-ab3a-444e-95ff-21633e93f4d5/plugins/VanguardRanks/ranks.yml) with benefits that don’t inflate money supply (QoL, access, reduced taxes/fees if supported).
  - Build a long ladder with costs matching your target economy pacing.
- Auction house integrity:
  - Configure CrazyAuctions with clear min/max prices and consider adding listing limits/fees (CrazyAuctions is limited here; if you require strong taxation and caps, you may need an alternative AH plugin).

### 3.7 Gems (Hard Currency) Implementation Path
- Choose one approach (based on current plugins):
  - Option 1 (recommended with current stack): Gems are an item currency
    - Use nightcore item definitions (see [nightcore currencies.yml](file:///v:/crafty/servers/04c9743c-ab3a-444e-95ff-21633e93f4d5/plugins/nightcore/currencies.yml)) to define a “Gem” item (custom tag/name/model).
    - Tebex sells packages that grant the Gem item.
    - Black market / crates / special menus charge Gems by consuming the item.
  - Option 2: Gems are a Skript virtual balance
    - Skript stores `%player%` gem balances; PlaceholderAPI exposes it; zMenu consumes it; Tebex grants it by dispatching commands.
  - If you need Gems to behave as a second Vault economy used everywhere: NEEDS ADDITIONAL.

### 3.8 Crates (Visible Loot Tables + Pity)
- Implement crate ladder:
  - Create crate definitions under `plugins/ExcellentCrates/crates/` and key definitions under `plugins/ExcellentCrates/keys/`.
  - Ensure previews are accessible and loot tables are visible to players.
- Configure pity:
  - Use the milestones system in [milestones.yml](file:///v:/crafty/servers/04c9743c-ab3a-444e-95ff-21633e93f4d5/plugins/ExcellentCrates/milestones.yml) as the pity/guarantee layer for rare rewards.
- Broadcast rare wins:
  - Enable appropriate broadcast announcements for high-tier rewards (configure per crate/reward).

### 3.9 Black Market (Rotating Stock)
- Implement as a GUI shop:
  - Build a zMenu inventory for “Black Market” that lists rotating items.
  - Use Skript to rotate stock on a schedule and charge Gems (item currency or Skript balance).

### 3.10 Events (KOTH/LMS/Tournaments)
- Minimal viable path with current plugins: PARTIAL
  - Script timed events with a protected arena area, temporary rules, and reward commands.
- Full-feature path: NEEDS ADDITIONAL
  - Add dedicated KOTH/LMS/tournament plugins and integrate rewards with your economy (money/gems/crates/hearts).

## 4) Top Gaps / Decisions To Lock In (Before Production Config Work)

- Hearts threshold policy: clarify whether elimination is at 0, 5, or “floor 5 + separate elimination”.
- Totem policy: either ban them or implement “totem pop still costs a heart”.
- Shop implementation choice: add a shop plugin vs build a controlled shop with zMenu + Skript.
- Gems representation: item-currency vs virtual currency vs second Vault economy.
- Death-loss consistency: disable /back-on-death and tune or remove graves to keep the loss loop intact.

