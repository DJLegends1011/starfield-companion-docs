# Section 6: Affinity Events — DRAFT v3
## (For "Aflin & DJLegend's Starfield NPC Template" — fills the Section 6 stub in the shared Google Doc)

*This section is specific to Companion/Follower NPCs. Affinity Events are Starfield's system for tracking how a companion feels about the player's actions. Where Skyrim followers mostly just exist, Starfield companions have opinions — every theft, firefight, and drink the player takes can move the relationship. If the Designer fills out this section, a Scripter should be able to implement the companion's entire reaction profile in the Creation Kit.*

*Everything in this section has been verified against the vanilla Papyrus sources (primarily `CompanionAffinityEventsScript.psc`) and a full data dump of vanilla affinity event forms. Where vanilla has bugs or dead events, they are flagged — don't write lines the game will never play.*

---

## How Affinity Works

When the player performs certain actions, visits locations, or makes dialogue choices, the game fires an **Affinity Event**. Your companion reacts on a **5-tier scale**:

| Reaction | Points | Use For... |
|----------|--------|------------|
| **LOVES** | +50 | Actions core to the NPC's identity/passions |
| **LIKES** | +25 | Actions the NPC approves of but aren't defining |
| **INDIFFERENT** | 0 | Neutral activities — still plays a line, just no points |
| **DISLIKES** | -15 | Actions that annoy or conflict with the NPC's values |
| **HATES** | -35 | Actions that violate the NPC's hard limits |

**INDIFFERENT is not "no reaction."** Most vanilla reactions are Indifferent — the companion still comments, the relationship just doesn't move. Flavor commentary lives here.

**Design tip:** Most events should land in LIKES/INDIFFERENT/DISLIKES. LOVES and HATES are powerful — save them for things that truly define your character. In vanilla, HATES is almost never used.

### Affinity Levels

As points accumulate, the companion progresses: **Neutral → Friendship → Affection → Commitment** (romance possible at the top, if applicable). Negative events can also feed an **Anger System** (Not Angry → Annoyed → Angry → Furious). Anger Level 2+ makes the companion confront the player and can end with them leaving — decide early which events are your companion's anger triggers.

### Types of Affinity Events

| Type | Prefix | Count in Vanilla | Purpose |
|------|--------|-----------------|---------|
| **Action Events** | `COM_Event_Action_*` | ~40 functional | React to player actions (steal, craft, combat, etc.) |
| **Comment Triggers** | `COM_Event_CommentTrigger_*` | ~319 | React to entering locations (cities, dungeons, landmarks) |
| **Dialogue Responses** | *(varies, often quest prefix)* | — | React to specific player dialogue choices |
| **WantsToTalk** | `COM_WantsToTalkEvent_*` | 12 | Story gate conversations the companion initiates |
| **Direct Events** | `COM_Event_Direct_*` | 4 per companion | Scripts manually award affinity (Likes/Dislikes/Loves/Hates) |

Action Events are the bread and butter — this section covers them event-by-event. The other types get shorter treatment at the end.

---

## Before You Write: How Action Events Actually Fire

All Action Events are sent by one vanilla quest script (`CompanionAffinityEventsScript`). Knowing its rules saves you from writing lines that rarely or never play:

1. **One shared cooldown throttles ALL action events.** After any action event fires, every other action event is suppressed for a global cooldown (`COM_ActionEventScriptFilter_CoolDownMinutes`). The player can steal five things in a row and get one line. Write lines knowing most triggers are silently eaten — variety matters more than volume for spammy actions.
2. **Events are dropped, not queued**, if the player is in dialogue, the companion is in a scene, or either is near an "important scene." No line plays and no points move.
3. **Events reach the active companion AND active Elite Crew.** You can give reactions to a crew-only NPC, not just full companions.
4. **Each event has an "Allow Repeated" flag** in the CK — some (Arrival) repeat forever, others fire once per playthrough.
5. **Reactions and lines are data, not script.** The designer picks a reaction tier per event and writes lines; the scripter sets them on the AffinityEvent form. No Papyrus needed for standard events.

---

## A) Action Events

*Format (following the Event Driven Dialogue sections): event, what actually triggers it, an example line, and a recommended minimum line count. Frequency notes tell you how often the player will realistically hear it. Fill in your companion's reaction tier for every event in the worksheet at the end of this section.*

### Survival & Environment

**Arrival** (`COM_Event_Action_Arrival`) : Fires when the companion exits the ship interior or the player walks into a new location — but not when entering a child location of somewhere already visited. Certain special locations bypass the cooldown entirely.
*"New place, same dust. Let's see what it's hiding."*
Recommend 10 minimum — this is the single most frequent event in the game.
— The game tracks whether the player recently disembarked, so conditions can distinguish "just landed" from "wandered in on foot" —

**Temperature High / Low** (`COM_Event_Action_TempHigh` / `TempLow`) : The planet has a VeryHot/VeryCold keyword. Checked on a 60-second poll after location changes, and only fires when the temperature *changes* from the last check.
*"My suit says it's fine out here. My suit is a liar."*
Recommend 2 minimum each.

**Gravity High / Low** (`COM_Event_Action_GravityHigh` / `GravityLow`) : Gravity above 1.5g / below 0.5g, polled after location changes, fires only on change.
*"Ugh. Every step on this rock counts double."*
Recommend 2 minimum each.

**Zero-G** (`COM_Event_Action_ZeroG`) : ⚠ **Effectively dead in vanilla.** The gravity check tests "below 0.5g" *before* "zero g," so zero gravity sends **GravityLow** instead — the ZeroG branch is unreachable. The script's own comments say quests that change gravity should send the event directly. Only write ZeroG lines if your mod will `Send()` the event itself; otherwise put your zero-g flavor into GravityLow.

**Swim** (`COM_Event_Action_Swim`) : Player enters deep water.
*"Whatever's in that water, you're about to meet it."*
Recommend 2 minimum.

**Environmental Hazard Warning** (`COM_Event_Action_EnvironmentalHazardWarning`) : Player's suit starts soaking environmental damage (entering a hazard zone before taking real damage).
Recommend 2 minimum.

**Environmental Damage — Radiation / Corrosive / Airborne / Thermal** (`COM_Event_Action_EnvironmentalDamage_*`) : Player actually takes that damage type.
*"You're smoking. Not in the good way. Move!"*
Recommend 1-2 minimum each (4 separate events).

**Jump From Height** (`COM_Event_Action_JumpFromHeight`) : Player falls far enough to take damage.
*"You know the boost pack has a button, right?"*
Recommend 2 minimum.

**Become Over-Encumbered** (`COM_Event_Action_BecomeOverEncumbered`) : Checked whenever the player picks something up while over carry weight.
*"You planning to open a shop, or can we drop some of this?"*
Recommend 3 minimum — players live in this state.

### Crime & Stealth

**Steal** (`COM_Event_Action_Steal`) : Player steals an item **and the companion has line-of-sight to the player**. If she can't see it, it never happened.
*"Smooth. Mostly. Don't quit your day job."*
Recommend 5 minimum.
— The line-of-sight rule is a gift: a thief character can be played honest in front of a disapproving companion —

**Pickpocket** (`COM_Event_Action_StealPickpocket`) : As above (line-of-sight required), but for pickpocketing.
Recommend 2 minimum.

**Pick Lock** (`COM_Event_Action_PickLock`) : Player successfully picks a lock that is not a terminal.
*"And that's why you bring a professional."*
Recommend 5 minimum — very frequent.

**Hack** (`COM_Event_Action_Hack`) : Player successfully breaks into a locked **terminal** (same trigger as Pick Lock, split by object type).
Recommend 3 minimum.

**Jail Complete** (`COM_Event_Action_JailComplete`) : Player finishes serving a jail sentence.
*"So. How was prison?"*
Recommend 1 minimum — rare, but a great character moment.

### Combat & Violence

**Civilian Combat / Civilian Killed** (`COM_Event_Action_CivilianCombat_Common` / `CivilianKilled_Common`) : Player attacks / kills someone the companion considers a civilian. ⚠ These are **not** sent by the standard action-event script — they route through the crime/anger system, and they double as the main vanilla **anger triggers**. Who counts as a "civilian" is defined by the Shared Crime Faction List on your companion's Personal Crime Faction, so a criminal companion can be set up to shrug at dead pirates.
*"That one wasn't shooting back, chief."*
Recommend 3 minimum each, plus anger-confrontation dialogue if these are your companion's line.

**Discharge Weapon** (`COM_Event_Action_DischargeWeapon`) : Player fires a weapon somewhere inappropriate — only in settlement-keyword locations, only with **no hostiles within 40 meters**, and melee weapons are excluded.
*"Great. Now everyone's looking at us."*
Recommend 2 minimum.

**Heal Companion** (`COM_Event_Action_HealCompanion`) : Player uses a healing item on the **current active companion**.
*"Huh. Thanks. Didn't think you'd noticed."*
Recommend 3 minimum — this is the game handing you a bonding moment; many vanilla companions LIKE or LOVE it.

### Crafting & Building

**Use Workbench** (`COM_Event_Action_UseWorkbench`) : ⚠ **DEAD IN VANILLA.** The form exists and is even assigned to all four vanilla companions, but the sending code is commented out (*"no longer being used. No companion lines were written for this"*, vanilla bug GEN-432521). It will never fire unless your mod re-sends it. Skip it, or budget scripter time.

**Craft Item** (`COM_Event_Action_Crafting_CreateItem`) : Player crafts anything at a bench.
Recommend 2 minimum.

**Mod Weapon / Mod Armor** (`COM_Event_Action_Crafting_Mod_Weapon` / `Mod_Armor`) : Player installs a weapon or armor mod (one trigger, split by item type).
*"Ooh. That'll leave a mark."*
Recommend 2 minimum each.

**Build Ship Module** (`COM_Event_Action_Crafting_CreateShipModule`) : Player modifies their ship in the ship builder.
*"New engines? Now you're speaking my language."*
Recommend 3 minimum — fires once per builder session, and ship opinions are strong character material.

**Build Outpost Module** (`COM_Event_Action_Crafting_CreateOutpostModule`) : Player places an outpost object (sent by the outpost beacon script).
Recommend 2 minimum.

### Exploration & Gathering

**Use Hand Scanner** (`COM_Event_Action_UseHandScanner`) : Player scans an object. Extremely frequent for survey-heavy players.
Recommend 2 minimum.

**Harvest Mineral / Plant** (`COM_Event_Action_Harvest_Mineral` / `Harvest_Plant`) : Player harvests from inorganic/organic flora.
Recommend 1-2 minimum each.

**Harvest Animal** (`COM_Event_Action_Harvest_Animal`) : Player takes an organic resource from a dead creature.
Recommend 1-2 minimum.

**Loot Corpse** (`COM_Event_Action_LootCorpse`) : Player takes anything (non-organic) from a dead body.
*"Not like they need it anymore."*
Recommend 3 minimum — constant in combat gameplay.

**Drop Useful Item** (`COM_Event_Action_DropUsefulItem`) : Player drops an item from a curated "useful items" keyword list onto the ground, outside combat.
*"You're really just gonna leave that there?"*
Recommend 2 minimum.

### Ship

**Ship Embark** (`COM_Event_Action_ShipEmbark`) : Companion boards the player's ship.
*"Home sweet home. Well. Home, anyway."*
Recommend 3 minimum.

**Boarding Enemy Ship** (`COM_Event_Action_ShipBoardingOther`) : Companion enters a ship flagged as a hostile boarding encounter. One of the most character-defining events in the game — a former pirate boards with excitement, a scientist with dread, a mercenary with professional calm. This single reaction tells the player more about your companion's past than a paragraph of backstory.
*"Breach and clear, just like old times!"*
Recommend 5 minimum.
— `ShipBeingBoarded` also exists as a form but is unused in vanilla —

### Social & Personal

**Drink Alcohol** (`COM_Event_Action_Consume_Alcohol`) : Player consumes a drink-keyword item.
*"Starting early? Respect."*
Recommend 3 minimum.

**Take Chems** (`COM_Event_Action_Consume_Chems`) : Player consumes illicit/non-medical drugs. (Trivia for scripters: the vanilla script property is named `Consume_Drugs`, but the form's Editor ID is `Consume_Chems` — don't let the mismatch confuse you.)
Recommend 3 minimum.

**Chem Addiction** (`COM_Event_Action_ChemAddiction`) : Player becomes addicted.
Recommend 1 minimum — rare, high-impact.

**Walk Around Naked** (`COM_Event_Action_WalkAroundNaked`) : Checked on location change — player is wearing neither clothing nor a spacesuit.
*"Bold fashion choice, chief."*
Recommend 2 minimum.

**Change Appearance** (`COM_Event_Action_Enhance_PlayerChangeAppearance`) : Fires when the player closes the appearance-change menu.
Recommend 1 minimum.

### Powers

**Use Starborn Power** (`COM_Event_Action_UseStarbornPower`) : Player casts a power with the Artifact keyword. How does a normal person feel watching their friend bend gravity? Fear, awe, envy, hunger — pick one and commit.
*"That's... yeah, I'm never getting used to that."*
Recommend 3 minimum.

### A note on quest-prefixed "action" events

Some events look global but aren't. Example: the charity event is actually `DialogueFCAkilaCity_COM_Event_Action_AAA_GivenCreditsToCharity` — it only fires when donating at one specific Akila City shelter. Events with a quest prefix are sent by that quest's scenes; you can add reactions to them, but don't design as if they fire everywhere.

---

## B) Comment Triggers

Location-based reactions — **~319 in vanilla**, covering cities, dungeons, quest locations, and points of interest. Each form includes a written description of what the companion is seeing, so the writer has context without visiting in-game.

You don't write all 319. Pick:
- **Major cities** your companion has opinions about (New Atlantis, Neon, Akila City, The Key…)
- **Locations tied to their backstory**
- **Landmarks** where their personality would shine

Comment Triggers allow repeats and default to INDIFFERENT (pure flavor, no points). Filter for `COM_Event_CommentTrigger_` in the CK for the full list.

---

## C) Dialogue Response Events

These fire when the companion reacts to a specific **dialogue choice** the player makes in conversation — typically at quest decision points.

**The key example (Legacy's End):** the Crimson Fleet questline ends with a cut-and-dry alignment test — side with the Fleet or with SysDef. Only a pirate-hearted companion would approve of the Fleet choice, which makes it the perfect place to show who your NPC really is.

Here's the surprise in the vanilla data: all four vanilla companions are **INDIFFERENT** to the final handover itself. Their affinity swings come from **what the player says along the way** ("I just want to do the right thing" → LIKES for some companions). Two lessons:
- Don't just plan reactions for quest *outcomes* — plan them for the dialogue *en route*. That's where personality lives.
- Vanilla left the big choice itself unreacted. If your companion takes a real stance on the outcome (via a quest event or a Direct event), you're already doing more with the moment than Bethesda did.

---

## D) WantsToTalk & Direct Events

**WantsToTalk events** make the companion approach the player for story-gate conversations, with per-event distance and cooldown settings. Plan one per story gate.

**Direct events** are your scripter's manual lever: each companion gets a set of four (`Likes` / `Dislikes` / `Loves` / `Hates`) that quest scripts fire to award affinity for moments no automatic event covers — personal quest choices, gifts, anything custom.

---

## E) Story Gates

Timed relationship milestones: affinity threshold reached → in-game timer runs → companion wants to talk → personal conversation scene → next gate begins. Plan **4-8 gates**, escalating from casual to personal:

| Gate | Suggested Focus |
|------|----------------|
| 1 | Surface-level bonding |
| 2 | Shared interests or experiences |
| 3 | First hints of backstory |
| 4 | Real backstory reveal |
| 5 | Vulnerability — fears and desires |
| 6 | Deep trust; romance available (if applicable) |

Each gate is a full conversation scene with branches. If your companion is romanceable, the final gate typically leads to the confession scene, with post-romance idle lines (recommend 10+) and breakup/rejection lines (recommend 5) to match.

---

## F) Worksheet: Your Companion's Affinity Profile

*Fill this out before writing dialogue. The reaction column is the design; the line counts are the writing budget. Leave the tier blank only if you're skipping the event entirely.*

**Master Action Event Table**

| Event | Reaction Tier | # Lines | Notes |
|-------|--------------|---------|-------|
| Arrival | | | |
| TempHigh / TempLow | | | |
| GravityHigh / GravityLow | | | |
| ZeroG *(needs custom Send)* | | | |
| Swim | | | |
| EnvironmentalHazardWarning | | | |
| EnvDamage (Rad/Corr/Air/Therm) | | | |
| JumpFromHeight | | | |
| BecomeOverEncumbered | | | |
| Steal | | | |
| StealPickpocket | | | |
| PickLock | | | |
| Hack | | | |
| JailComplete | | | |
| CivilianCombat / CivilianKilled | | | anger trigger? |
| DischargeWeapon | | | |
| HealCompanion | | | |
| ~~UseWorkbench~~ *(dead)* | — | — | skip or re-send via script |
| Crafting_CreateItem | | | |
| Mod_Weapon / Mod_Armor | | | |
| CreateShipModule | | | |
| CreateOutpostModule | | | |
| UseHandScanner | | | |
| Harvest (Mineral/Plant/Animal) | | | |
| LootCorpse | | | |
| DropUsefulItem | | | |
| ShipEmbark | | | |
| ShipBoardingOther | | | |
| Consume_Alcohol | | | |
| Consume_Chems | | | |
| ChemAddiction | | | |
| WalkAroundNaked | | | |
| Enhance_PlayerChangeAppearance | | | |
| UseStarbornPower | | | |

**Quick profile:**

- Actions my companion **LOVES** (pick 3-5): ______
- Actions my companion **LIKES** (pick 5-10): ______
- Actions my companion **DISLIKES** (pick 3-5): ______
- Actions my companion **HATES** (0-2, and mean it): ______
- **Anger triggers** (what makes them confront the player): ______
- **Story gate count:** ___ | **Romanceable:** Yes / No | **Orientation:** ______

**Faction stances** (informs Dialogue Responses and Comment Triggers in faction territory):

| Faction | Stance | Notes |
|---------|--------|-------|
| United Colonies | | Authority, military, law and order |
| Freestar Collective | | Frontier justice, independence |
| Ryujin Industries | | Corporate power, espionage |
| Crimson Fleet | | Piracy, freedom, moral grey |
| Constellation | | Exploration, curiosity |
| House Va'ruun | | Religious zealotry |

---

*Sources: `CompanionAffinityEventsScript.psc`, `AffinityEvent.psc`, `CompanionAffinityScript.psc` (Starfield Data/Scripts/Source), plus a CK data dump of all vanilla affinity event forms. Fire conditions, thresholds (gravity 1.5g/0.5g, 40m hostile radius), the shared cooldown, the dead UseWorkbench event, and the unreachable ZeroG branch are all verified against those sources.*
