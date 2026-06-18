# Omar's Asymmetric Game Framework

A Roblox framework for building 1-vs-many asymmetric horror games (one "Killer," several "Survivors" — think the genre Dead by Daylight popularized). It provides the character/ability backbone and a working round loop, and leaves presentation (UI, menus, polish) to whoever builds on top of it.

## What this actually is

This is **not** a finished game. It's the systems layer underneath one:

- An ability system (cooldowns, slots, swappable "support" abilities, a registry so abilities can be referenced by string ID).
- Character base classes for Killer and Survivor (health/movement, stamina, status effects, animations, voicelines).
- A round/lobby state machine, killer selection, map loading, and networking to drive a match from "waiting for players" to "results screen."
- A client-side input layer that turns key presses into server-validated ability calls.

What it deliberately does **not** include: any UI (HUDs, menus, cooldown displays, character select screens), any actual abilities or characters (only the base classes they'd be built from), animations/sounds (only placeholder asset ID slots), or real maps (only the folder structure the round logic expects). The round loop in `RoundService` is functional but is a skeleton too — it currently builds killer and survivor loadouts full of empty slots (`AbilityLoadout.new(nil, nil, nil, nil, nil)`) because no concrete characters have been registered yet, and the lobby spawn point is left unset. Treat it as the wiring you plug your own characters, abilities, and front-end into, not as something you can ship as-is.

## Folder layout

```
ReplicatedStorage
  Abilities/   AbilityBase, AbilityLoadout, AbilityRegistry
  Classes/     BaseCharacter, BaseKiller, BaseSurvivor, CharacterRegistry
  Network/     Remotes
  Systems/     GameStateManager, KillerSelector

ServerScriptService
  Services/    RoundService, MapService

StarterPlayerScripts
  Controllers/ InputController
```

## How it all works together

1. **Matchmaking.** `RoundService` runs a loop through `GameStateManager` states: `Lobby → WaitingForPlayers → Countdown → InGame → PostGame`, broadcasting each transition to clients over the `StateChanged` remote.
2. **Role assignment.** When a round starts, `KillerSelector` picks whoever has been Killer the fewest times (random tiebreak), and `RoundService` builds a `BaseKiller` for them and a `BaseSurvivor` for everyone else, each wrapping an `AbilityLoadout`. Survivors' support-slot choices (picked during the lobby phase via the `SwapSupport` remote) are pulled in here.
3. **World setup.** `MapService` clones a random map, finds its `Spawns/Killer` and `Spawns/Survivors` folders, and teleports everyone in.
4. **Input.** On the client, `InputController` listens for raw key/mouse input, maps it to a slot-name string (e.g. `Q`), and fires it to the server over the `UseAbility` remote. The client does no prediction — it just asks the server and waits.
5. **Server-side handling.** `RoundService` receives the slot name, finds the player's `BaseKiller`/`BaseSurvivor` object, and calls `:HandleInput(inputName)`. Each character class has its own `INPUT_MAP` (e.g. Killer's `Q` → `PRIMARY_2`) that translates the raw key into an ability slot, which is forwarded to `AbilityLoadout:ActivateSlot`, which calls the actual `AbilityBase:Activate` → `OnActivate`.
6. **Per-frame updates.** `RoundService` runs one `Heartbeat` loop for the whole round and calls `:Update(dt)` on every active character, which in turn updates stamina, locomotion animations, and any active abilities (`AbilityLoadout:UpdateAll`).
7. **Win conditions & cleanup.** Every second, `RoundService` checks survivor health states for a win/loss, then ends the round, shows results, destroys character wrapper objects and the map clone, and teleports everyone back to the lobby.

The key idea throughout: **abilities and characters don't know about round rules**, and **round logic doesn't know ability internals** — they only talk through the small public methods each layer exposes (`Activate`, `HandleInput`, `Update`, `Destroy`). That's what makes the next two sections possible without touching the core files.

## Adding more keybinds

Keybinds flow through three places that all have to agree on the same slot name string:

1. **`AbilityLoadout`** — add the new slot to the `SlotKey` type and to the `_slots` table in `AbilityLoadout.new`.
2. **A character's `INPUT_MAP`** (in `BaseKiller`, `BaseSurvivor`, or your own subclass) — map a raw input name (e.g. `"G"`) to the new slot name (e.g. `"PRIMARY_6"`).
3. **`InputController`'s `KEY_MAP`/`MOUSE_MAP`** — map the actual `Enum.KeyCode` or `Enum.UserInputType` to that same raw input name.
4. **`RoundService`'s `VALID_INPUTS` table** in `RoundService.Init` — add the raw input name here too, or the server will silently drop it.

Nothing else needs to change. The slot's ability still gets assigned normally through `AbilityLoadout.new(...)`.

## Adding more characters

There are no concrete characters in this framework yet — only the base classes. To add one:

1. **Write the abilities first.** Each ability is a subclass of `AbilityBase` (give it `OnActivate`/`OnDeactivate`/`OnUpdate` overrides) and gets registered with `AbilityRegistry.Register("MyAbilityId", MyAbilityClass)` so it can be looked up by string ID (used for support-slot swapping and networking).
2. **Subclass `BaseKiller` or `BaseSurvivor`** for the new character. Override `.new()` to set character-specific defaults (health, speed, `TerrorRadius`, `StoreConfig`, `AnimationConfig`, etc. — anything you don't set falls back to `BaseCharacter.Defaults`), and construct its `AbilityLoadout` with instances of that character's abilities in the fixed primary slots.
3. **Register the character class** with `CharacterRegistry.Register("MyCharacterId", MyCharacterClass, "Killer")` (or `"Survivor"`) so character-select UI (once built) can list it and pull display info via `CharacterRegistry.GetCharacterInfo`.
4. **Wire it into `RoundService`.** Replace the placeholder `AbilityLoadout.new(nil, nil, nil, nil, nil)` calls in `AssignRoles` with logic that looks up the player's chosen character (via `CharacterRegistry.Get`) and instantiates that class instead of the raw `BaseKiller`/`BaseSurvivor`.

Survivors' two support slots are filled separately and don't need a character subclass at all — any ability registered with `IsSwappable = true` is eligible, and players choose between them with the `SwapSupport` remote before the round starts.

---

# Script-by-script reference

## `AbilityBase` (Abilities)

The base class every ability inherits from. Construction takes a config table (`Name`, `Description`, `Cooldown`, `IsSwappable`, `IsPassive`) and sets up internal state: `_lastUsed` (so it's ready immediately on creation) and `_isActive`.

It exposes cooldown helpers — `IsReady()`, `GetCoolDownRemaining()`, `GetCoolDownFraction()` (0 = ready, 1 = just used, handy for driving a UI cooldown radial) — and three lifecycle hooks meant to be overridden by subclasses: `OnActivate`, `OnDeactivate`, `OnUpdate`. The lifecycle itself is handled by the non-overridden methods `Activate` (checks `IsReady`, stamps `_lastUsed`, flips `_isActive`, calls `OnActivate`), `Deactivate` (calls `OnDeactivate` if currently active), and `Destroy`. Subclasses should never need to touch `Activate`/`Deactivate` directly — they just implement the `On*` hooks.

## `AbilityLoadout` (Abilities)

Holds one character's full set of equipped abilities across seven fixed slots: `PRIMARY_1`–`PRIMARY_5` (fixed per character class, not swappable) and `SUPPORT_1`/`SUPPORT_2` (swappable, survivor-only by convention). It's a thin dictionary wrapper with the slot-keyed table as its only real state.

Its job is to route slot-name calls to the right ability instance: `ActivateSlot`/`DeactivateSlot` call through to the ability's own `Activate`/`Deactivate`, and `UpdateAll` calls `OnUpdate` on every currently-active ability each frame. `SwapSupport` lets a support slot be replaced at runtime — it asserts the new ability has `IsSwappable = true`, destroys whatever was previously in that slot, and replaces it. `ClearSupport` empties both support slots. `Destroy` cleans up every slot's ability when the character is removed.

## `AbilityRegistry` (Abilities)

A lookup table mapping string IDs to ability classes, used instead of passing class references around directly (useful for networking, since you can send a short string instead of a reference). `Register(id, class)` asserts no duplicate ID exists; `Get(id)` asserts the ID exists and returns the class; `GetAll()` returns the whole table.

`GetSwappable()` is worth calling out specifically: to find out which registered abilities are swappable, it constructs a **temporary throwaway instance of every single registered ability class** (with a mock `{Name = "Temp", ...}` config), checks `probe.IsSwappable`, then immediately destroys the probe. This works because `IsSwappable` is only ever known at instance-construction time, but it's flagged in the source itself as a hack that should probably be revisited — it runs ability constructors purely for introspection.

## `BaseCharacter` (Classes)

The shared foundation for both Killer and Survivor. It deliberately knows nothing about game rules (win conditions, rounds, matchmaking) — that's left to `RoundService`. What it owns: the Roblox model reference, the `AbilityLoadout`, status effects, stamina, locomotion animation state, and a big nested defaults table.

**Defaults & config merging.** `BaseCharacter.Defaults` defines fallback values for five config categories — `StoreConfig` (shop price/description/preview), `AnimationConfig` (animation IDs per action, all placeholder `rbxassetid://0` until filled in), `ChaseThemeConfig` (killer-only music layers), `VoicelineConfig` (random voice line pools per event), and `GameplayConfig` (health, speed, stamina rates). `BaseCharacter.new` takes whatever config the caller (or a subclass) provides and recursively fills in anything missing via `applyDefaults`, so subclasses only need to specify the fields they actually want to override.

**Setup quirks.** On construction it deletes the model's default `Animate` script and stops any animation tracks Roblox auto-started, since this framework drives all animation itself through `PlayAnimation`/`StopAnimation` (which load and cache `AnimationTrack`s keyed by name, and auto-detect looping/priority based on whether the key contains "idle"/"walk"/"run" vs. "ability"/"interaction").

**Other systems:** `UpdateLocomotion` runs a tiny state machine each frame to pick the right movement animation (idle/walk/run, prefixed with "Hurt" if `IsHurt()` returns true — which `BaseCharacter` itself always returns `false` for; `BaseSurvivor` overrides it). Stamina drains while sprinting and regenerates otherwise, force-stopping a sprint at zero. Status effects are named, timed buffs/debuffs with `OnApply`/`OnRemove` callbacks — reapplying a name refreshes its duration rather than stacking. `UseAbility` and the per-frame `Update` simply delegate to the loadout. `Destroy` tears down connections, status effects, the loadout, and loaded animation tracks.

## `BaseKiller` (Classes)

Extends `BaseCharacter`. Sets killer-flavored defaults (20/40 walk/sprint speed, 666 health, all five `PRIMARY` slots usable) and adds `TerrorRadius`, plus `KilledSurvivors`/`EscapedSurvivors` counters meant to be incremented from overridable hooks (`OnSurvivorKilled`, `OnSurvivorEscaped`) once that gameplay logic exists. Its `INPUT_MAP` binds mouse-1 and Q/E/F/R to all five primary slots. `HandleInput` looks up the slot for a raw input name and forwards to `UseAbility` — there's no extra gating here (a killer can always act, unlike survivors).

## `BaseSurvivor` (Classes)

Extends `BaseCharacter`. Only uses two primary slots (Q/E) plus the two support slots (F/R), since survivors are meant to have fewer, more utility-oriented kit pieces. Adds a `HealthState` state machine (`Healthy → Injured → Dead/Escaped`) with a `SPEED_MODIFIERS` table that scales movement speed per state (e.g. 85% while injured, 0 once dead or escaped). `SetHealthState` updates speed and fires the overridable `OnHealthStateChanged` hook. `IsHurt()` is overridden here to actually mean something (`true` while `Injured`), feeding back into `BaseCharacter`'s locomotion animation logic.

`CanUseAbility()` gates ability use behind health state and an `IsTasking` flag (meant for busy/interacting states like repairing something), and `HandleInput` checks this before doing anything. `SwapSupportAbility` is the runtime entry point for changing a support slot — it accepts either a raw ability instance or a string ID (resolved through `AbilityRegistry`), and is meant to be called during the lobby phase, before the round locks in.

## `CharacterRegistry` (Classes)

Mirrors `AbilityRegistry` but for whole characters: `Register(id, class, charType)` stores a class alongside whether it's `"Killer"` or `"Survivor"`. `GetCharacterInfo(id)` is the interesting one — like `AbilityRegistry.GetSwappable`, it constructs a throwaway probe instance (using a mock player table to dodge `BaseCharacter`'s assertions) purely to read its config values, then destroys it. It returns a flat info table — name, description, health, speed, raw `InputMap`, and pre-stringified versions of the store/animation/gameplay configs (`serializeTable` recursively flattens nested tables into indented text) — meant to be dropped straight into UI labels without the UI needing to know the class's internals. `GetAllInfo()` returns this for every registered character, e.g. for a character-select screen.

## `Remotes` (Network)

The single source of truth for every `RemoteEvent` used by the framework: `UseAbility`, `AssignRole`, `StateChanged`, `SyncCooldown`, `SwapSupport`, `ShowResults`, `SelectCharacter`. `Remotes.Init()` (server-only, called once at startup) creates a `Remotes` folder in `ReplicatedStorage` and instantiates any remote from that list that doesn't already exist. `Remotes.Get(name)` is used by both server and client to fetch a remote by name, waiting up to 10 seconds for it to exist (covers the client racing the server's `Init`). Any new remote event needs to be added to `REMOTE_NAMES` here — nothing else creates remotes.

## `GameStateManager` (Systems)

A small, generic state machine — not asymmetric-game-specific, just a reusable transition guard. States: `Lobby`, `WaitingForPlayers`, `Countdown`, `InGame`, `PostGame`. `VALID_TRANSITIONS` whitelists which state can move to which, so `Transition(newState)` silently warns and refuses an invalid jump rather than letting state get corrupted. Every successful transition fires a `BindableEvent` (exposed as `.OnStateChanged`) with the new and old state, so any other system (UI, other services) can react without needing a direct reference to whatever is driving the state — in this framework, that's `RoundService`. `Is(state)` is a convenience boolean check.

## `KillerSelector` (Systems)

Pure selection logic, no side effects beyond what's passed in. `SelectKiller(players, killerHistory)` counts how many times each player has already been killer (from a `killerHistory` table keyed by `UserId`, defaulting missing entries to 0), finds the minimum count, collects everyone tied at that minimum as candidates, and picks one at random — so over many rounds, killer turns even out instead of staying random/repetitive. `RecordKillerRound` increments a player's count after they've played killer; `RoundService` is responsible for calling this and for persisting/clearing `killerHistory` (it clears a player's entry on `PlayerRemoving`). `DebugStats` formats the current counts as a printable string for debugging.

## `MapService` (Services)

Handles map selection and teleportation. Expects a `Maps` folder in `Workspace` where each child is a self-contained map template. `PickRandomMap` shuffles the available maps (Fisher–Yates, via the shared `shuffleArray` helper) and returns one — note it returns a reference to the *template*, so the caller (`RoundService`) is responsible for `:Clone()`-ing it before use and destroying the clone after the round. `GetAvailableMaps` just lists the `Maps` folder's children.

`TeleportPlayers(map, killer, survivors)` expects each map to contain a `Spawns` folder with `Killer` and `Survivors` subfolders of spawn parts; it asserts these exist and that there are enough survivor spawn points for however many survivors there are, then moves each player's `HumanoidRootPart` to a (shuffled, for survivors) spawn point with a small vertical offset to avoid spawning inside the floor. `TeleportToLobby` does the same for returning everyone to a single lobby `CFrame` after the round ends.

## `RoundService` (Services)

The orchestrator — owns the full round lifecycle and is the only script that's allowed to know about all the other subsystems at once. It holds the module-level `GameStateManager` instance, the `killerHistory` table, the currently active `BaseKiller`/`BaseSurvivor` wrapper objects, and a `pendingSupportChoices` table tracking each player's chosen support abilities before a round starts.

`matchmakingLoop` is the outermost loop: it sits in `WaitingForPlayers` until `MIN_PLAYERS` is met, then a `Countdown` that resets back to waiting if the player count drops below the minimum mid-countdown, then calls `runRound()`. `runRound` clones a map, picks a killer via `KillerSelector`, splits the rest into survivors, calls `AssignRoles` to build the actual character wrapper objects (this is the function with the placeholder `nil`-filled loadouts mentioned earlier — it's where real character selection would plug in), teleports everyone in, transitions to `InGame`, and starts a `Heartbeat` connection driving every active character's `Update(dt)`. It then polls once a second for a win condition (`checkWinConditions`, based on survivor `HealthState`/`IsDead` counts) or for the round timer (`ROUND_DURATION`) to expire, and on either calls `endRound` (broadcasts results, waits `POSTGAME_TIME`) followed by `cleanUp` (destroys character wrapper objects and the map clone, teleports everyone back to the lobby).

`RoundService.Init()` is the entry point: it calls `Remotes.Init()`, wires up the three server-side remote listeners — `UseAbility` (validates the game is `InGame` and the input name is in a hardcoded `VALID_INPUTS` whitelist, then forwards to the right character's `HandleInput`), `SwapSupport` (only allowed outside `InGame`, validates the ability ID exists via `AbilityRegistry.Get` inside a `pcall`, and stores the choice for use when the round starts) — wires up cleanup on `PlayerRemoving`, and finally `task.spawn`s `matchmakingLoop`.

Things to note when extending this file: `VALID_INPUTS` here must stay in sync with `InputController`'s key maps (see "Adding more keybinds" above); `LOBBY_SPAWN_CFRAME` is currently `nil` and needs a real value before `cleanUp`'s `TeleportToLobby` call will work; and `AssignRoles` always builds raw `BaseKiller`/`BaseSurvivor` instances rather than looking up a player's chosen character class — that's the main thing to change once `CharacterRegistry` actually has entries in it.

## `InputController` (Controllers, client-only)

Deliberately the dumbest script in the framework — it does no game logic, only translates raw input into a slot-name string and fires it at the server, trusting the server to validate and apply everything (so a small amount of input latency is traded for the server staying fully authoritative; no client-side prediction happens here).

`KEY_MAP` and `MOUSE_MAP` map `Enum.KeyCode`/`Enum.UserInputType` values to input-name strings (`"Q"`, `"E"`, `"F"`, `"R"`, `"M1"`) — these must match whatever a character's `INPUT_MAP` expects on the server side. `InputController.Init()` requires the `Remotes` module and connects to `UIS.InputBegan`, ignoring input the game engine already consumed (`gameProcessed`) or input while `_enabled` is false, and otherwise fires `UseAbility` with the mapped name. `InputController.SetEnabled` exists so something else (the missing UI layer, presumably) can toggle input on/off — e.g. disabling ability input outside of `InGame`. There's a commented-out block at the bottom showing the intended wiring for that (auto-disabling input based on the `StateChanged` remote) that's currently switched off for testing and would need to be re-enabled for a real build.
