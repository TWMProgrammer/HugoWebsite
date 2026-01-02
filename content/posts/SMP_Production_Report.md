# SMP Production Readiness Report

## 1. Executive Summary
This report summarizes the server’s current configuration and provides a production action plan. The focus is performance, security, economy stability, grief prevention, and player experience.

**Key Findings:**
*   **Security:** `online-mode=false` and a Discord bot token is stored in plaintext; both are production blockers.
*   **Performance:** `view-distance=12` and `simulation-distance=12` are high for production; `max-chained-neighbor-updates=1000000` is a redstone lag/DOS risk.
*   **Anti-Xray:** Paper anti-xray is present but currently disabled.
*   **Economy:** DailyRewards/CrazyAuctions are mostly default and currently inject high-value items (diamonds) without an economy model.
*   **Nametag Systems:** TAB scoreboard teams are enabled and PvPManager also uses scoreboard teams, which commonly conflicts.

---

## 2. Core Server Configuration Plan (Current → Target)
This section combines the most important current values with the recommended production targets.

*   **`server.properties`**
    *   `online-mode`: Current `false` → Target `true`.
    *   `view-distance`: Current `12` → Target `7–8`.
    *   `simulation-distance`: Current `12` → Target `5–6`.
    *   `difficulty`: Current `easy` → Target `normal` or `hard`.
    *   `max-chained-neighbor-updates`: Current `1000000` → Target lower to reduce redstone lag/DOS risk.
    *   `rate-limit`: Current `0` → Target non-zero to reduce reconnect spam.
    *   `enforce-secure-profile`: Current `true` → Target depends on your chat policy (especially if you block chat reporting).

*   **`spigot.yml`**
    *   `world-settings.default.merge-radius.item`: Current `0.5` → Target `2.5–4.0` if item entities are a performance problem.
    *   `world-settings.default.merge-radius.exp`: Current `-1.0` → Target `4.0–6.0` if XP orbs are a performance problem.
    *   `entity-activation-range`: Current animals/villagers `32` → Target lower if MSPT is high in farms.

*   **`config/paper-world-defaults.yml`**
    *   `anticheat.anti-xray.enabled`: Current `false` → Target `true` if your economy depends on ore scarcity.
    *   `anticheat.anti-xray.engine-mode`: Current `1` → Target stronger mode if anti-xray is enabled and economy depends on ores/ingots/crates.

---

## 3. Installed Plugin Inventory

The following plugins have been identified from the `plugins/*.jar` inventory (not just folders). Some folders exist without a matching jar and may be leftovers.

### **Core & Management**
*   **Essentials / EssentialsDiscord / EssentialsDiscordLink:** Core server management, discord integration.
*   **LuckPerms:** Permission management.
*   **PlaceholderAPI:** Placeholders for other plugins.
*   **ProtocolLib:** Library for packet manipulation.
*   **PacketEvents:** Library used by some plugins for packet-level features.
*   **Tebex:** Web store integration.
*   **FastAsyncWorldEdit (FAWE):** World editing and schematic handling.
*   **Chunky:** World pre-generation.
*   **InvSeePlusPlus:** Inventory management/viewing.
*   **ViaVersion:** Multi-version client compatibility.
*   **Vault:** Economy/permissions/chat abstraction layer.

### **Security & Protection**
*   **CoreProtect:** Grief rollback and logging.
*   **LibertyBans:** Ban and punishment management.
*   **Protect:** Server security/firewall.
*   **CommandBlocker:** Command blocking/hiding.
*   **AntiPopup:** Prevents chat reporting popups.
*   **AntiHealthIndicator:** Hides health indicators.

### **Economy & Trade**
*   **AxTrade:** Secure player-to-player trading.
*   **CrazyAuctions:** Global auction house.
*   **ExcellentCrates:** Crate keys and rewards.
*   **DailyRewards:** Daily login bonuses.
*   **PvPManager:** Combat logging and PvP toggling.
*   **nightcore:** Economy/currency engine dependency for some plugins.

### **Gameplay & Mechanics**
*   **BetterRTP:** Random teleportation.
*   **AxGraves:** Death chests/graves.
*   **TerraformGenerator:** Custom world generation.
*   **ClearLaggEnhanced:** Entity and item lag clearing.
*   **TpaGui:** GUI for teleport requests.

### **Chat & Visuals**
*   **TAB:** Tab list and scoreboard customization.
*   **DecentHolograms:** Holographic displays.
*   **ajLeaderboards:** Leaderboards (Vault stats, mined blocks, kills, etc.).
*   **SimpleScore:** Scoreboard plugin (installed jar present).

---

## 4. Plugin Configuration Analysis

### **Priority 1: Critical Gameplay & Stability**
*   **BetterRTP:**
    *   Current: `Respect.WorldGuard=false` and `Respect.GriefPrevention=false`.
    *   *Important Note:* WorldGuard is not available for 1.21.9, and BetterRTP does not list Protect as a supported “Respect” integration. If you use Protect for regions, keep RTP away from protected hubs (spawn) by using a minimum radius, a separate RTP world, or an RTP center far from protected regions.
*   **ClearLaggEnhanced:**
    *   Current: whitelist includes villagers, armor stands, item frames, paintings, and multiple vehicle types.
    *   *Risk to review:* Clearing systems can still break farms and item transport if tuned aggressively. Your current config is not overly destructive by default.
*   **PvPManager:**
    *   Current: `Use Scoreboard Teams: true` is enabled.
    *   *High likelihood of conflict:* TAB has `scoreboard-teams.enabled: true`. Decide which plugin owns scoreboard teams to avoid flickering/nametag overrides.
*   **VeinMiner:**
    *   Current: `blocks.json` is empty, but `groups.json` already defines an `Ores` group using tags (`#c:ores` with `#minecraft:pickaxes`).
    *   *Recommended improvement:* Add an additional group for logs (and optionally clay/sand) only if you want VeinMiner to cover more than ores.

### **Priority 2: Economy & Features**
*   **EssentialsDiscord:**
    *   Current: `guild` is set to all zeros and the staff channel ID is all zeros.
    *   **CRITICAL SECURITY ISSUE:** A Discord bot `token` is present in plaintext in the config. Treat it as compromised: rotate it immediately and do not store secrets in files that might be shared.
*   **Essentials (worth.yml / server sell prices):**
    *   **File:** `plugins/Essentials/worth.yml`
    *   **What it does:** Controls what items can be sold (typically via `/sell` and `/worth`). Items not listed cannot be sold.
    *   **Current state:** A large worth list is present and allows selling many items, including high-value economy drivers (e.g., diamonds/diamond blocks/ores) and some blocks that are usually non-survival or restricted (e.g., bedrock). The file also contains legacy-style material keys and data-value sections (e.g., `wool: '0': 20`, `leaves:`), which may not map cleanly on modern versions.
    *   **Production risks:**
        *   Server selling becomes a primary currency faucet and can quickly inflate balances, especially if ores/gems are sellable.
        *   Legacy material/data-value entries can behave unexpectedly (wrong items priced, or items not sellable) after version jumps.
        *   Farmable items (cactus/sugarcane, etc.) can dominate the economy if server buy prices are too generous.
    *   **Configuration plan (recommended):**
        *   Decide whether you want server-selling at all. If your economy is meant to be player-driven (shops/AH), keep `worth.yml` minimal.
        *   Whitelist only low-impact, farmable “utility” outputs (e.g., common crops, basic mob drops) and set payouts low enough that player shops/AH remain competitive.
        *   Disable or remove all “wealth drivers” from server selling unless intentionally abundant: ores/ingots, diamonds/emeralds, blocks of metals/gems, redstone/lapis, etc.
        *   Remove entries for restricted/non-survival blocks (e.g., bedrock) so they cannot be converted into money if obtained via exploits.
        *   Normalize entries to modern material naming (and avoid data values) to reduce version-mapping ambiguity; verify each key by testing `/worth <item>` and `/sell` with sample stacks.
*   **DailyRewards:**
    *   Current: config is mostly default and currently issues iron/gold/diamonds/emeralds at fixed intervals.
    *   **Production risk:** Injecting diamonds/emeralds on timers will destabilize any survival economy (shops, AH, crates, mining value).
*   **CrazyAuctions:**
    *   Current: mostly default settings (price caps, minimums, durations).
    *   **Production risk:** Without listing limits, taxes, anti-dupe monitoring, and item restrictions, AH becomes the economy’s primary exploit surface.
*   **AxTrade:**
    *   Current: core settings are fine, but it is permissive by default (same-IP trade allowed, broad allowed items).
    *   **Recommendation:** Disable same-IP trading and blacklist high-risk items if your server sees alt abuse or dupes.
*   **DecentHolograms:** Default range (48 blocks) is fine, but ensure holograms don't clutter spawn.

---

## 5. Security & Anti-Cheat
*   **Production blockers:**
    *   `online-mode=false` breaks identity guarantees and makes punishments/logs unreliable.
    *   Secrets stored in plaintext (Discord token, management secret) must be rotated and handled safely.
*   **Protect (WorldGuard alternative for 1.21.9):**
    *   Protect is a WorldGuard-like region protection plugin that provides “areas” with flags (entry/exit, damage, hunger, redstone, liquid flow, physics, greetings/farewell messages, etc.) and is designed with a robust API and permission packs.
    *   Use Protect for spawn/hub regions and staff-managed protected areas while staying on 1.21.9.
    *   Expect some differences from WorldGuard: integrations and “flag ecosystems” may not map 1:1, so verify compatibility for plugins that rely on WorldGuard hooks.
*   **CoreProtect:**
    *   Current: `use-mysql: false` (SQLite). This can be fine for small servers, but for long retention and high activity, move to MariaDB/MySQL.
*   **LibertyBans:**
    *   Current: configured for `server-type: ONLINE` while the server is `online-mode=false`. These must match the actual authentication mode.
*   **AntiPopup:**
    *   `block-chat-reports: true` is enabled. Ensure this aligns with your `enforce-secure-profile` decision.
*   **Protection plan (recommended):**
    *   Standardize on Protect for staff regions (spawn, event zones, shops) and document the default flag policy (what’s allowed/denied by default).
    *   Add a player-claims layer if you want players to self-protect bases (separate from staff regions), then decide how it should interact with Protect-protected hubs.
    *   Decide an anti-cheat approach. Paper anti-xray helps against basic x-ray, but it is not a full anti-cheat solution.

---

## 6. Performance Tuning
*   **Chunky:** Use this to pre-generate the world border *before* opening.
    *   Command: `/chunky start`
*   **Spark:** Enabled in Paper config. Use it for MSPT profiling and entity hot spots.
*   **Timings:** Disabled in Paper config. Prefer Spark for analysis.
*   **Highest-impact changes (recommended):**
    *   Lower `view-distance` and `simulation-distance`.
    *   Enable Paper anti-xray if your economy depends on ore scarcity.
    *   Reduce `max-chained-neighbor-updates` to mitigate redstone lag machines.
    *   Ensure only one plugin manages scoreboard teams (TAB vs PvPManager vs SimpleScore).

---

## 7. Theming & Gameplay Customization
To bring the server to a "Production" standard, we need a consistent visual identity and streamlined player experience.

### **Visual Identity (Theme)**
*   **Style:** Modern, Clean, High-Contrast.
*   **Primary Colors:** Gold (`&6`) and Dark Gray (`&8`).
*   **Accent Colors:** Yellow (`&e`) for highlights, Red (`&c`) for errors.

#### **1. MOTD (Message of the Day)**
*   **File:** `plugins/Essentials/motd.txt`
*   **Current State:** Default Essentials motd lines (functional, not themed).
*   **Action:** Replace with a consistent banner (optional).
    ```text
    &8&m-----------------------------------------------------
      &6&lPROJECT SMP &7- &e&lSEASON 1
         &7&oForge your destiny in a new world!
    
      &6&l> &fType &e/help &fto get started
      &6&l> &fStore: &estore.projectsmp.net
    &8&m-----------------------------------------------------
    ```

#### **2. Chat Formatting (EssentialsChat)**
*   **File:** `plugins/Essentials/config.yml` (Search for `chat:` section)
*   **Current State:** Global format only: `{DISPLAYNAME}: &F{MESSAGE}`. Group formats are not configured.
*   **Action:** Define distinct formats for groups (optional, but recommended for clarity).
    ```yaml
    chat:
      radius: 0
      format: '&8[&f{GROUP}&8] &7{DISPLAYNAME}&8: &f{MESSAGE}'
      group-formats:
        owner: '&8[&4OWNER&8] &c{DISPLAYNAME}&8: &c{MESSAGE}'
        dev: '&8[&4DEV&8] &c{DISPLAYNAME}&8: &c{MESSAGE}'
        admin: '&8[&cADMIN&8] &c{DISPLAYNAME}&8: &c{MESSAGE}'
        mod: '&8[&3MOD&8] &b{DISPLAYNAME}&8: &f{MESSAGE}'
        vip: '&8[&bVIP&8] &b{DISPLAYNAME}&8: &f{MESSAGE}'
        default: '&8[&7Member&8] &7{DISPLAYNAME}&8: &f{MESSAGE}'
    ```

### **Player Onboarding & Help**
The server currently uses **zMenu** for interactive help, which is excellent.

#### **1. Interactive Menus (zMenu)**
*   **Status:** **Active & Configured.**
*   **File:** `plugins/zMenu/inventories/main_menu/main_menu.yml`
*   **Current State:** The menu includes Basics, Homes, Economy, Kits, Rules, RTP, and AH.
*   **Redirects:** `/help`, `/?`, and `/ehelp` correctly open this menu (verified in `commands.yml` and `zMenu/commands/commands.yml`).
*   **Recommendation:** Ensure the `basics_menu` and `rules` items inside the main menu are populated with up-to-date information.

### **Permissions Hierarchy (LuckPerms)**
Analysis of `luckperms-2026-01-02-00-29.json` confirms the following hierarchy:

*   **Structure:**
    1.  **Owner** (Inherits `dev`)
    2.  **Dev** (Inherits `admin`) - *Note: Prefix has a typo "Developper"*
    3.  **Admin** (Inherits `mod`)
    4.  **Mod** (Inherits `vip`)
    5.  **VIP**
    6.  **Default** (Implicit)

*   **Action Items:**
    1.  **Fix Typo:** Update the Dev group prefix.
        ```bash
        lp group dev meta setprefix "&4&l&o&n[Developer]&r "
        ```
    2.  **Verify Inheritance:** Ensure `vip` inherits from `default` so VIPs get basic survival commands.
        ```bash
        lp group vip parent add default
        ```

### **Security & Command Cleanup**
We will prioritize **Permissions** to hide commands, using `CommandBlocker` only as a fallback.

#### **1. Permission-Based Hiding (Primary)**
*   **Goal:** Prevent default players from seeing or using technical commands.
*   **Action:** Negate specific permission nodes for the `default` group.
    ```bash
    # Hide plugin lists and version info
    lp group default permission set bukkit.command.plugins false
    lp group default permission set bukkit.command.version false
    lp group default permission set bukkit.command.help false
    lp group default permission set minecraft.command.help false
    lp group default permission set minecraft.command.me false
    lp group default permission set minecraft.command.tell false
    
    # Hide specific plugin commands (examples)
    lp group default permission set luckperms.info false
    lp group default permission set essentials.help false
    ```

#### **2. CommandBlocker (Secondary/Fallback)**
*   **File:** `plugins/CommandBlocker/config.yml`
*   **Action:** Use this ONLY for commands that cannot be hidden via permissions (e.g., complex aliases or hardcoded plugin commands).
*   **Recommendation:** Keep the list minimal to reduce overhead.
    ```yaml
    Mode: BLACKLIST
    Message: "&8[&cSystem&8] &7Unknown command."
    Commands:
      - /icanhasbukkit
      - /?
      - /about
    ```

---

## 8. Production Configuration Plans
### 8.1 Launch Gating Checklist (Before Public Release)
*   Authentication policy finalized (`online-mode` decision + UUID mode alignment across plugins).
*   Secrets handled safely (rotate Discord token, remove plaintext secrets from shared storage).
*   Grief protection in place (claims/regions) and spawn protected.
*   Economy model defined (currency sources/sinks, crate odds, AH limits/taxes, anti-inflation).
*   Performance budget set (target MSPT and player count) with Spark baselines taken.

### 8.2 Backups & Recovery
*   Backup cadence: at least daily (worlds + `plugins/` configs + permissions exports).
*   Retention: short-term frequent (e.g., 7–14 daily) + long-term weekly/monthly snapshots.
*   Offsite storage: at least one external location (not on the same disk/host).
*   Restore drill: documented steps and a quarterly test restore to a staging copy.

### 8.3 Economy Stability Plan
*   Remove/replace high-value item injections (diamonds/emeralds) from DailyRewards unless they are intentionally abundant.
*   Define currency sinks (teleport costs, repair fees, crate opening costs, auction taxes).
*   Define listing limits (per-player concurrent listings, min/max prices, restricted items).
*   Decide whether crates are cosmetic-only or economy-impacting, then tune odds accordingly.

### 8.4 Anti-Grief & Moderation Workflow
*   Pick a claiming/protection solution and standardize it (player claims + staff regions).
*   Define staff response flow (CoreProtect lookup/rollback policy, ban escalation templates).
*   Decide log retention and storage (SQLite vs MariaDB) for CoreProtect based on expected activity.

### 8.5 Performance & World Management
*   World border defined and Chunky pregen executed before launch.
*   Entity-clearing and limiters tuned to preserve player builds/farms.
*   Single source of truth for nametags/teams (TAB vs PvPManager vs scoreboard plugins).
