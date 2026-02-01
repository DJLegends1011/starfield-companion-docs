# Starfield Companion Creation Guide

**FOR EXTERNAL USE** - Generic step-by-step guide for creating ANY companion from scratch.

This guide applies to any companion character and is based on Andreja's vanilla implementation. For character-specific guidance, refer to that companion's design document (e.g., `YourCompanion_GDD.md`).

**How to Use This Guide:**
1. Read the entire guide to understand companion architecture
2. Gather your character's design documentation
3. Work through phases sequentially (1-13)
4. Refer to **[ADVANCED FEATURES](#advanced-features-optional)** for optional polish
5. Replace all `YourName` placeholders with your character's name

---

## TABLE OF CONTENTS

1. [Overview](#1-overview)
2. [Phase 1: Actor Setup](#phase-1-actor-setup)
3. [Phase 2: VoiceType Setup](#phase-2-voicetype-setup)
4. [Phase 3: Quest Setup](#phase-3-quest-setup)
5. [Phase 4: Quest Aliases](#phase-4-quest-aliases)
6. [Phase 5: Dialogue - Combat & Detection](#phase-5-dialogue---combat--detection)
7. [Phase 6: Dialogue - Misc (Hello/Goodbye)](#phase-6-dialogue---misc-hellogoodbye)
8. [Phase 7: Dialogue - Custom Topics (Affinity)](#phase-7-dialogue---custom-topics-affinity)
9. [Phase 8: Scenes - System (Follow/Wait/Dismiss)](#phase-8-scenes---system-followwaitdismiss)
10. [Phase 9: Scenes - Story Gates](#phase-9-scenes---story-gates)
11. [Phase 10: Scenes - Romance & Commitment](#phase-10-scenes---romance--commitment)
12. [Phase 11: Greeting/Top Level Scenes](#phase-11-greetingtop-level-scenes)
13. [Phase 12: Companion Reaction Integration](#phase-12-companion-reaction-integration)
14. [Phase 13: Testing](#phase-13-testing)

---

## 1. OVERVIEW

### What Makes a Full Companion

A Starfield companion consists of:

1. **Actor** - The NPC with CompanionActorScript
2. **VoiceType** - For dialogue filtering
3. **Personal Quest** (COM_Companion_*) - Manages relationship
4. **Dialogue** - Combat, detection, misc, affinity events
5. **Scenes** - Story conversations, system commands, reactions

### Time Estimate

| Phase | Complexity | Notes |
|-------|------------|-------|
| Actor Setup | Low | Copy existing, modify |
| VoiceType | Low | Simple creation |
| Quest Setup | Medium | Multiple components |
| Dialogue | High | 200+ lines minimum |
| Scenes | High | 20+ scenes |
| Reaction Integration | Very High | 200-400 vanilla scenes |

---

## PHASE 1: ACTOR SETUP

### Step 1.1: Create the Actor

1. Open Creation Kit
2. Find an existing companion (e.g., `Companion_Andreja`)
3. Duplicate it
4. Rename to `Companion_YourName`

### Step 1.2: Configure Actor Flags

| Flag | Setting | Purpose |
|------|---------|---------|
| Essential | ☑ | Cannot be killed |
| Unique | ☑ | Only one instance |
| Doesn't affect stealth meter | ☑ | Won't blow cover |
| Ignore Friendly Hits | ☑ | Won't turn hostile |

### Step 1.3: Attach Required Scripts

| Script | Purpose |
|--------|---------|
| `CompanionActorScript` | Core companion functions |
| `CompanionAffinityScript` | Affinity tracking |
| `CompanionCrimeResponseScript` | Crime reactions |
| `COM_CREW_GiveItemActorScript` | Item transfer |

### Step 1.4: Configure CompanionActorScript Properties

```
SQ_Companions → SQ_Companions quest
COM_Affinity → COM_Affinity ActorValue
COM_PreventStoryGateScenes → COM_PreventStoryGateScenes keyword
COM_CompanionQuest → Your COM_Companion_* quest (create later)
COM_CompanionID → Unique GlobalVariable for this companion
COM_PQ_TxtReplace_QuestName → Activator with quest name
```

### Step 1.5: Set Appearance

- Configure face, body, hair, etc.
- Set race, sex
- Add inventory/outfit

### Step 1.6: Keywords Tab

Keywords are tags that enable specific behaviors and system hooks.

**Required Companion Keywords:**

| Keyword | Purpose |
|---------|---------|
| `Crew_CrewTypeCompanion` | **Essential** - Marks as full companion (not just crew) |
| `NoPickpocket` | Player cannot pickpocket companion |
| `PlayerCanStimpak` | Player can heal companion with stimpaks |
| `TeammateDontUseAmmoKeyword` | Infinite ammo (doesn't consume) |
| `SQ_Followers_SandboxWhilePlayerLoiters` | Sandbox behavior when player idles |
| `SQ_Followers_TeleportToShipWithPlayer` | Teleports with player to ship |
| `TeammateReadyWeapon_DO` | Weapon ready behavior |

**Optional Keywords:**

| Keyword | Purpose |
|---------|---------|
| `ActorSocialImmune` | Immune to certain social/dialogue triggers |
| `NoTerrormorphMindControl` | Immune to terrormorph mind control |
| `AnimArchetypeYourName` | Custom animation set (if created) |
| `ArmorTypeApparelOrNakedBody` | Armor system flag |

### Step 1.7: AI Data Tab

AI Data controls combat and social behavior.

**Standard Companion Settings:**

| Attribute | Typical Value | Meaning |
|-----------|---------------|---------|
| **Aggression** | Aggressive | Will attack enemies on sight |
| **Confidence** | Foolhardy | Never flees, fights to the death |
| **Assistance** | Helps Allies | Will help friendly NPCs in combat |
| **Morality** | No Crime / Any Crime | Crime willingness (character-dependent) |
| **Combat Style** | Default | Inherited from template |
| **Energy** | 50 | Base energy level |

**Morality Options:**

- `No Crime` - Won't commit crimes (Sarah, Barrett, Andreja)
- `Any Crime` - Will help with crimes (appropriate for pirate/criminal characters)

**Flee Settings:**
- Generally unchecked for companions (they don't flee)
- Flee Distance, Safe Timer, Chance of Diversion: Use global defaults

### Step 1.8: AI Packages Tab

AI Packages control NPC behavior schedules and routines.

**Package Lists:**

| List Type | Value |
|-----------|-------|
| **Default Package List** | `DefaultMasterPackageList` |
| **Spectator Override** | NONE |
| **Observe Dead Body Override** | NONE |
| **Guard Warn Override** | NONE |
| **Combat Override** | NONE |

**Custom Packages to Create:**

| Package | Purpose |
|---------|---------|
| `YourName_RecruitmentPkg` | Behavior during recruitment quest |
| `YourName_SandboxDefault` | Default idle behavior |
| `YourName_HomePkg` | Behavior at "home" location |

**Notes:**
- Most behavior comes from `DefaultMasterPackageList`
- Custom packages handle specific quest moments
- "Any/Any/Any" schedule means no restrictions

### Step 1.9: Inventory Tab

Controls starting gear, outfits, and death loot.

**Required Items:**

| Item | Notes |
|------|-------|
| **Starting Weapon** | Custom or generic weapon |
| **Default Outfit** | `Outfit_YourName` - NOTPLAYABLE versions |
| **Spacesuit Outfit** | `Outfit_Spacesuit_YourName` |

**Outfit Settings:**

| Setting | Value |
|---------|-------|
| **Default Outfit** | `Outfit_YourName` |
| **Space Suit Outfit** | `Outfit_Spacesuit_YourName` |
| **Power Armor Furniture** | NONE |

**Death Item:**
- Create `LLD_Companion_YourName` leveled list
- Or use existing faction list (e.g., `LLD_CrimsonFleet` for former pirates)

**Outfit Record Pattern:**
- `Clothes_YourName_NOTPLAYABLE` - Casual clothes
- `Spacesuit_YourName_NOTPLAYABLE` - Unique spacesuit
- `Spacesuit_YourName_Backpack_NOTPLAYABLE` - Matching backpack
- `Spacesuit_YourName_Helmet_NOTPLAYABLE` - Matching helmet

### Step 1.10: Spell List / Perks Tab

This is where companion skills/perks are assigned. **No spells, just perks.**

**Required Perk:**

| Perk ID | Rank | Purpose |
|---------|------|---------|
| `Companion_BasicPerk` | 1 | **Required** - Base companion functionality |

**Skill Perks (Choose Based on Character):**

| Perk ID | Description |
|---------|-------------|
| `Crew_Piloting` | Ship piloting skill |
| `Crew_Stealth` | Stealth skill |
| `Crew_Ballistics` | Ballistic weapon proficiency |
| `Crew_Lasers` | Laser weapon proficiency |
| `Crew_ParticleBeams` | Particle beam proficiency |
| `Crew_Shotgun_Certification` | Close quarters specialist |
| `Crew_Ship_Weapons_Ballistic` | Ship ballistic weapons |
| `Crew_Ship_Weapons_Energy` | Ship energy weapons |
| `Crew_Theft` | Theft/pickpocket skill |

**Example - Combat Specialist:**
```
Crew_Ballistics - Rank 3
Crew_Shotgun_Certification - Rank 2
Crew_Piloting - Rank 2
Companion_BasicPerk - Rank 1
```

**Example - Stealth Specialist (Andreja):**
```
Crew_Stealth - Rank 4
Crew_ParticleBeams - Rank 3
Crew_Ship_Weapons_Energy - Rank 2
Crew_Theft - Rank 1
Companion_BasicPerk - Rank 1
```

### Step 1.11: Factions Tab

Factions control NPC relationships, crime reporting, and group behavior.

**Required Factions:**

| Faction | Rank | Purpose |
|---------|------|---------|
| `AvailableCompanionFaction` | -1 | Marks as recruitable companion |
| `CurrentCompanionFaction` | -1 | Currently following player (changes when active) |
| `PlayerAllyFaction` | 0 | Allied with player |

**Create New Faction:**

| Faction | Purpose |
|---------|---------|
| `COM_PersonalCrimeFaction_YourName` | **NEW** - Personal crime tracking |

Set as **Assigned Crime Faction** in the Factions tab.

#### Personal Crime Faction Setup

When creating `COM_PersonalCrimeFaction_YourName`, configure these tabs:

**Crime Tab Settings:**

| Setting | Value | Purpose |
|---------|-------|---------|
| **Track Crime** | ☑ Checked | Enables crime tracking for this companion |
| **Attack on Sight** | ☑ Checked | Companion attacks criminals on sight |
| **Arrest** | ☐ Unchecked | Companions don't arrest, just react |

**Ignore Crimes Against Non-Members:**

These checkboxes control which crimes the companion *ignores* when committed against NPCs NOT in the shared crime list:

| Crime Type | Andreja (Standard) | Notes |
|------------|-------------------|-------|
| Murder | ☐ Unchecked | Reacts to murder |
| Assault | ☐ Unchecked | Reacts to assault |
| Pickpocket | ☑ Checked | Ignores pickpocketing |
| Smuggling | ☑ Checked | Ignores smuggling |
| Stealing | ☑ Checked | Ignores theft |
| Trespass | ☑ Checked | Ignores trespassing |
| Piracy | ☑ Checked | Ignores piracy |

**For Cass (Criminal Background):** Consider checking MORE boxes (ignoring more crimes) since she has `Morality: Any Crime`.

**Shared Crime Faction List:**

Dropdown selection that determines who counts as a "civilian." Options include:
- `FactionSharedCrimeList` - Standard civilian list (most companions use this)
- Various other FormLists for specific factions

The `CompanionCrimeResponseScript` checks this list when `CivilianKilled` events fire.

**Voices Tab:**

Links the crime faction to the companion's VoiceType for dialogue/reactions.

| Field | Value |
|-------|-------|
| **Valid Voices** | `NPCFYourName` (your companion's VoiceType) |

This is how the `CompanionCrimeResponseScript` knows which companion this faction belongs to.

**Optional Factions (Character-Dependent):**

| Faction | When to Use |
|---------|-------------|
| `ConstellationFaction` | If Constellation member |
| `CrimsonFleetFaction` | If Fleet affiliated (rank -1 for former) |
| `FreestarRangersFaction` | If Freestar affiliated |
| `UCMilitaryFaction` | If UC affiliated |
| `LodgeInvisibleDoorsFaction` | If they should access Lodge |

**Faction Rank Meanings:**
- **Rank -1:** Inactive/potential member
- **Rank 0:** Active member
- **Rank 1+:** Higher standing

### Step 1.12: Stats Tab (VoiceType)

**Critical Setting:**

| Field | Value |
|-------|-------|
| **Voice Type** | `NPCFYourName` (create in Phase 2) |

This links to:
- Scene dialogue (GetIsVoiceType conditions)
- Generic follower barks
- Combat voice lines
- Idle chatter

### Step 1.13: Template Tab

**Recommended Template:**

Use `Companion_Crew_Elite_BalanceDataTemplate` for stats.

This provides:
- Appropriate health scaling
- Combat style defaults
- Level scaling

**To Customize Combat:**
- Override Combat Style at actor level
- Or create new template with custom combat style

---

## PHASE 2: VOICETYPE SETUP

### Step 2.1: Create VoiceType

1. In CK, go to Character → VoiceType
2. Create new: `NPCFYourName` or `NPCMYourName`
3. Set appropriate flags

### Step 2.2: Add to CompanionVoicesList

1. Find FormList `Default_CompanionVoicesList`
2. Add your new VoiceType to the list
3. This enables companion reaction filtering

### Step 2.3: Assign to Actor

1. Open your Actor
2. Set VoiceType to your new VoiceType

---

## PHASE 3: QUEST SETUP

### Step 3.1: Create Personal Quest

1. Create new quest: `COM_Companion_YourName`
2. Set as Start Game Enabled (or trigger via story)

### Step 3.2: Attach COM_CompanionQuestScript

Go to Scripts tab, add `COM_CompanionQuestScript`

### Step 3.3: Configure Script Properties

**Autofill Properties:**
```
SQ_Followers → SQ_Followers quest
SQ_Companions → SQ_Companions quest
COM_StoryGatesCompleted → ActorValue
COM_CurrentStoryGateTimerExpired → ActorValue
COM_PersonalQuest_Started → ActorValue
COM_PersonalQuest_Finished → ActorValue
COM_CommitmentQuest_Started → ActorValue
COM_AngerCoolDownTimerExpired → ActorValue
COM_AngerSceneCompleted → ActorValue
COM_AngerSecondChances → ActorValue
COM_AffinityLevel_1_Friendship → GlobalVariable
COM_AffinityLevel_2_Affection → GlobalVariable
COM_AffinityLevel_3_Commitment → GlobalVariable
COM_IsRomantic → ActorValue
COM_HasBeenRomantic → ActorValue
COM_CommitmentPossible → ActorValue
COM_CommitmentDesired → ActorValue
COM_IsCommitted → ActorValue
COM_CommitmentRefusedPermanently → ActorValue
```

**Custom Properties:**
```
Alias_Companion → Your companion alias (create in Phase 4)
PersonalQuest → Your personal quest (create later)
CommitmentQuest → Your commitment quest (create later)
WantsToTalkAnnouncementScene → Scene (create in Phase 9)
WantsToTalkConditionForm → COM_CND_WantsToTalk_YourName
StoryGateData → Array of StoryGateDatum (gate numbers + timers)
CompanionCheckPerk → Perk for dialogue conditions
```

### Step 3.4: Create Story Gate Data

Example configuration:
```
Gate 1: 1800 seconds (30 min)
Gate 2: 3600 seconds (1 hour)
Gate 3: 5400 seconds (1.5 hours)
Gate 4: 7200 seconds (2 hours)
Gate 5: 3600 seconds (1 hour)
Gate 6: 1800 seconds (30 min)
```

---

## PHASE 4: QUEST ALIASES

### Step 4.1: Create Required Aliases

In your `COM_Companion_YourName` quest, Quest Aliases tab:

| Alias Name | Type | Fill Type |
|------------|------|-----------|
| `YourName` | Ref | UniqueActor 'Companion_YourName' |
| `ActiveCompanion` | Ref | External: SQ_Companions.ActiveCompanion |
| `PlayerShipCrewMarker` | Ref | External: SQ_PlayerShip.HomeShipCrewMarker |
| `playerShipInteriorLocation` | Loc | External: SQ_PlayerShip.playerShipInteriorLocation |
| `WaitLocation` | Ref | Forced: Your waiting marker |
| `Player` | Ref | UniqueActor 'Player' |

### Step 4.2: Configure External Alias

For `ActiveCompanion`:
```
Fill Type: ● External Alias Ref
           ☑ (Linked)
           Quest: SQ_Companions
           Alias: ActiveCompanion
Flags: ☑ Optional
```

### Step 4.3: Link Alias to Script

Set `Alias_Companion` property to your companion alias.

---

## PHASE 5: DIALOGUE - COMBAT & DETECTION

### Step 5.1: Combat Tab Subtypes

Create dialogue for each combat subtype:

**Ground Combat:**
- `(AvoidThreat)` - 5+ lines
- `(BleedOut)` - 3+ lines
- `(CrippledLimb)` - 3+ lines
- `(Death)` - Death sounds
- `(EnemyKilled)` - 5+ lines
- `(MeleeAttack)` - 3+ lines
- `(OnHit)` - 5+ lines
- `(PowerAttack)` - 3+ lines
- `(Taunt)` - 5+ lines
- `(ThrowGrenade)` - 3+ lines

**Spaceship Combat (20+ subtypes):**
- `(SpaceshipEnterCombat)`
- `(SpaceshipLeaveCombat)`
- `(SpaceshipEnemyEngineDown/Repaired)`
- `(SpaceshipEnemyShieldsDown/Repaired)`
- `(SpaceshipEnemyWeaponsDown/Repaired)`
- `(SpaceshipEngineDown/Repaired)`
- `(SpaceshipShieldsDown/Repaired)`
- `(SpaceshipWeaponsDown/Repaired)`
- etc.

### Step 5.2: Detection Tab Subtypes

Create dialogue for state transitions:

| Subtype | When |
|---------|------|
| `(NormalToAlert)` | Heard something |
| `(NormalToCombat)` | Enemy spotted |
| `(NormalToLost)` | Sensed but can't find |
| `(AlertToCombat)` | Confirmed threat |
| `(AlertToNormal)` | False alarm |
| `(CombatToLost)` | Lost enemy mid-fight |
| `(CombatToNormal)` | Fight over |
| `(LostToCombat)` | Re-found enemy |
| `(LostToNormal)` | Gave up |

---

## PHASE 6: DIALOGUE - MISC (HELLO/GOODBYE)

### Step 6.1: Create Condition Forms

Create ConditionForms for each tier:
- `COM_CND_DIAL_YourName_Hello_Angry`
- `COM_CND_DIAL_YourName_Hello_Friendship`
- `COM_CND_DIAL_YourName_Hello_Affection`
- `COM_CND_DIAL_YourName_Hello_Committed`
- (Same pattern for Goodbye, WaitingForInput)

### Step 6.2: Hello Lines (4 tiers, 10+ each)

**Hello_Anger:**
```
Lines when companion is angry at player
```

**Hello_Friendship:**
```
Lines at Friendship level
```

**Hello_Affection:**
```
Lines at Affection level
```

**Hello_Romance:**
```
Lines when in romantic relationship
```

### Step 6.3: Goodbye Lines (4 tiers, 10+ each)

Same tier structure as Hello.

### Step 6.4: WaitingForPlayerInput (3 tiers)

**WFI_Positive:**
```
Patient, friendly waiting lines
```

**WFI_Neutral:**
```
Neutral waiting lines
```

**WFI_Negative:**
```
Impatient, annoyed waiting lines
```

---

## PHASE 7: DIALOGUE - CUSTOM TOPICS (AFFINITY)

### Step 7.1: Affinity Event Responses

For each affinity event in the game (100+), create a response line.

**Topic Structure:**
```
Editor ID: COM_YourName_Event_Action_*
Conditions: Event-specific
Response: Character-appropriate reaction
```

### Step 7.2: Keyword Topics

Create special topics:
- `COM_SleepTopic_PlayerWakesUp` - Waking up near player
- `COM_SleepTopic_WakeUpInPlayersBed` - Romance wake-up
- `COM_TradeTopic` - Trading dialogue

---

## PHASE 8: SCENES - SYSTEM (FOLLOW/WAIT/DISMISS)

### Step 8.1: System Scenes Required

| Scene | Purpose |
|-------|---------|
| `COM_YourName_System_PickupScene` | Recruitment |
| `COM_YourName_System_DismissScene` | Dismissal |
| `COM_YourName_System_FollowScene` | Resume following |
| `COM_YourName_System_WaitScene` | Wait command |
| `COM_YourName_System_WantsToTalk_NotHere` | Delay conversation |

### Step 8.2: Dismiss Scene Structure

**Player line (shared):**
```
<<Shared>> I think it's time we went our separate ways.
```

**Response groups:**
```
[GROUP] COM_YourName_Dismiss_Positive
  - High affinity/romance responses (reluctant)

[GROUP] COM_YourName_Dismiss_Negative
  - Low affinity/angry responses (eager)
```

### Step 8.3: Scene Scripts

At end of scenes, call appropriate functions:
```papyrus
; In PickupScene end fragment:
COM_CompanionQuestScript.PickupSceneEnded()

; In DismissScene end fragment:
COM_CompanionQuestScript.DismissSceneEnded()

; In WaitScene end fragment:
COM_CompanionQuestScript.WaitSceneEnded()

; In FollowScene end fragment:
COM_CompanionQuestScript.FollowSceneEnded()
```

---

## PHASE 9: SCENES - STORY GATES

### Step 9.1: Story Gate Scenes

Create scenes for each gate:
- `COM_YourName_Story_IntroScene01`
- `COM_YourName_Story_IntroScene02`
- `COM_YourName_Story_SG01Scene`
- `COM_YourName_Story_SG02Scene`
- `COM_YourName_Story_SG03Scene`
- `COM_YourName_Story_SG04Scene`
- `COM_YourName_Story_SG05_QuestIntro`
- `COM_YourName_Story_SG06Scene`
- `COM_YourName_Story_SG07_CommitmentIntroScene`

### Step 9.2: Gate Scene Structure

Each gate scene should:
1. Have conditions based on `COM_StoryGatesCompleted` and timer
2. Progress the relationship narrative
3. Call `StoryGateSceneCompleted()` at end

### Step 9.3: Wants to Talk Scene

Simple one-liner scene:
```
"Hey, got a minute? There's something I want to talk about."
```

Set as `WantsToTalkAnnouncementScene` property.

---

## PHASE 10: SCENES - ROMANCE & COMMITMENT

### Step 10.1: Flirt Scenes

- `COM_YourName_TopLevel_Flirt`
- Multiple phases with rotating options (FlirtChoiceMax)
- Call `FlirtSceneEnded()` at end

### Step 10.2: Romance Scenes

- `COM_YourName_System_RomanceScene`
- `COM_YourName_TopLevel_RomanceScene`
- Confession/relationship initiation
- Call `MakeRomantic()` when romance starts

### Step 10.3: Commitment Scenes

- `COM_YourName_Story_SG07_CommitmentIntroScene`
- `COM_YourName_TopLevel_Commitment_Break`
- Call `StartCommitmentQuest()` for ceremony
- Call `BreakCommitment()` for breakup

### Step 10.4: Anger Scene

- `COM_YourName_System_AngerScene`
- Confrontation when companion is angry
- Speech challenge option
- Call `AngerSceneCompleted()` at end

---

## PHASE 11: GREETING/TOP LEVEL SCENES

### Step 11.1: Add to Greeting Scenes List

In Greeting/Top Level tab, add your scenes with appropriate:
- Index (priority order)
- Conditions (ConditionForms)

### Step 11.2: Add to Top Level Scenes List

Add all TopLevel scenes:
- System scenes (Dismiss, Follow, Wait, Pickup)
- Relationship scenes (Flirt, Romance, GiveItem)

### Step 11.3: Set Scene Conditions

Each scene needs conditions to appear:
- `IsTrueForConditionForm` for most
- `GetIsAliasRef` for alias-specific
- `GetInFaction` for faction checks

---

## PHASE 12: COMPANION REACTION INTEGRATION

### Step 12.1: Identify Reaction Scenes

Search for scenes with "CompanionReaction" or "Companion" alias.

**Key quests with reactions:**
- MQ103 (One Small Step)
- MQ104A (Into the Unknown)
- MQ105 (The Old Neighborhood)
- All faction quests
- Major story beats

### Step 12.2: Add Your Lines

For each reaction scene:

1. Open the scene
2. Find dialogue action with `CompanionAlias`
3. Add new Topic Info with:
   - Condition: `GetIsVoiceType NPCFYourName == 1.00`
   - Response: Character-appropriate line
   - Affinity Event (optional)

### Step 12.3: Example

**MQ104A Temple Reaction:**
```
Condition: GetIsVoiceType 'NPCFYourName' == 1.00
Line: "[AF][C] Your character's reaction to first temple..."
```

---

## PHASE 13: TESTING

### Step 13.1: Basic Functionality

- [ ] Can recruit companion
- [ ] Companion follows correctly
- [ ] Can dismiss companion
- [ ] Can tell companion to wait
- [ ] Can resume following

### Step 13.2: Dialogue Testing

- [ ] Combat barks play
- [ ] Detection barks play
- [ ] Hello/Goodbye lines work
- [ ] Affinity events trigger

### Step 13.3: Affinity Testing

- [ ] Affinity increases on positive actions
- [ ] Affinity decreases on negative actions
- [ ] Level transitions work
- [ ] Story gates trigger correctly

### Step 13.4: Romance Testing

- [ ] Flirt options appear
- [ ] Romance can be initiated
- [ ] Commitment quest works
- [ ] Breakup works

### Step 13.5: Edge Cases

- [ ] Companion doesn't break when dismissed mid-quest
- [ ] Switching companions works
- [ ] Locked-in companion check works
- [ ] Death handling works (if applicable)

---

## QUICK REFERENCE: CRITICAL GOTCHAS

### Soft Lock Prevention

1. **Always check `COM_PQ_LockedInCompanion`** before `SetRoleActive()`
2. **Call `SetRoleAvailable()`** if player declines, NOT nothing
3. **Call `EvaluatePackage()`** after `SetRoleActive()`

### Common Mistakes

1. Forgetting to add VoiceType to `Default_CompanionVoicesList`
2. Not setting External Alias as "Linked"
3. Missing `[C]` conditions on VoiceType-filtered lines
4. Forgetting to call scene end functions

### Files to Back Up

- Actor record
- Quest record
- All dialogue
- All scenes
- VoiceType
- FormList edits

---

## ADVANCED FEATURES (OPTIONAL)

These systems add polish but aren't required for a functional companion. Implement after core companion is working.

---

### A1. LOCATION COMMENTARY QUESTS

Trigger ambient conversations when visiting specific locations.

**Quest Type:** Script Event

**Naming Convention:**
```
COM_Companion_YourName_Convo_LocationName##
```

**Examples from Andreja:**
| Quest ID | Location | Purpose |
|----------|----------|---------|
| `COM_Companion_Andreja_Convo_Neon01` | Neon | Hometown memory |
| `COM_Companion_Andreja_Convo_NewAtlantis01` | New Atlantis | City commentary |
| `COM_Companion_Andreja_Convo_RedMile01` | Red Mile | Arena reaction |

**Setup:**
1. Create new quest with Script Event type
2. Set location trigger conditions
3. Add dialogue scene that plays on location enter
4. 7 Users typically (quest + conditions + dialogue)

**Example Location Combinations:**
- Companion's hometown - Character nostalgia/reflection
- Restricted area - Character avoidance (faction-specific)
- Recruitment location - Callback to first meeting
- Key story location - Relevant character commentary

---

### A2. COMPANION-TO-COMPANION BANTER

> **⚠️ NOTE:** Actor Dialogue Events have NOT been reverse-engineered yet. This section is based on observation only - needs further CK research before implementation.

Two-way conversations between companions when both are present.

**Quest Type:** Actor Dialogue Event

**Naming Convention:**
```
COM_Companion_YourName_Convo_OtherCompanion##
```

**Examples from Andreja:**

*Andreja initiates:*
| Quest ID | With | Topic |
|----------|------|-------|
| `COM_Companion_Andreja_Convo_Barrett01` | Barrett | Philosophy |
| `COM_Companion_Andreja_Convo_SarahMorgan01` | Sarah | Leadership |
| `COM_Companion_Andreja_Convo_SamCoe01` | Sam | Family |

*Others initiate with Andreja:*
| Quest ID | Initiator | Topic |
|----------|-----------|-------|
| `COM_Companion_Barrett_Convo_Andreja01` | Barrett | Response |
| `COM_Companion_SarahMorgan_Convo_Andreja01` | Sarah | Response |

**Setup:**
1. Create Actor Dialogue Event quest
2. Set conditions for both companions present
3. Create two-actor scene
4. Both companions need aliases

**Example Companion Interaction Types:**
| Type | Example | Topic |
|------|---------|-------|
| Attracted | Companion A → Companion B | "I like them" |
| Friendly | Companion A → Companion B | "Good to have you around" |
| Hostile | Companion A → Companion B | "Keep away from me" |
| Mocking | Companion A → Companion B | "That's ridiculous" |
| Neutral | Companion A → Companion B | "We work well together" |

Refer to character design document for specific interaction dynamics.

---

### A3. LODGE AMBIENT CONVERSATIONS

Ambient banter at Constellation HQ (or other hub locations).

**Quest Location:** `DialogueUCTheLodge`

**Priority:** 15 (lower than companion quests)

**Examples:**
| Quest ID | Participants |
|----------|--------------|
| `DialogueUCTheLodge_Convo03_AndrejaMatteo` | Andreja + Matteo |
| `DialogueUCTheLodge_Convo14_AndrejaSam` | Andreja + Sam |

**Customization Tip:**
If character isn't faction-affiliated, replace Lodge with relevant hubs:
- Bars and cantinas (character origin location)
- Ship interior (if character lives aboard)
- Character's home base or sanctuary

---

### A4. GIFT SYSTEM (COM_CREW_GiveItemActorScript)

Companions periodically give player items as gifts.

**Script:** `COM_CREW_GiveItemActorScript` (already attached to Actor)

**Key Properties:**

| Property | Type | Purpose |
|----------|------|---------|
| `GiveItemList` | LeveledItem | Items to give (filter: `GiveItemList`) |
| `COM_CREW_GiveItemAvailableCooldownDays` | GlobalVariable | Days between gifts |
| `COM_CREW_GiveItemReminder` | Keyword | Dialogue subtype for reminder |

**How It Works:**
1. On location change, script checks if cooldown passed
2. If conditions met, sets `COM_CREW_GiveItem_HasItems = 1`
3. Companion says reminder line via `SayCustom()`
4. `GiveItems()` adds from leveled list to player inventory

**Gift List Design Tips:**
Leveled items should reflect character personality:
- Practical items (based on character skills)
- Faction-appropriate items (based on character background)
- Currency (universal)
- Equipment that matches character's profession
- Consumables that match character preferences

---

### A5. QUEST PRIORITY REFERENCE

Higher priority = runs first when multiple quests compete.

| Priority | Usage |
|----------|-------|
| 0 | System quests (speech challenges, shared lines) |
| 15 | Lodge/hub ambient conversations |
| 30 | Commitment/romance quests |
| 70 | Main companion dialogue quests |
| 99 | Personal quests (highest priority) |

---

### A6. EVENT TYPES

> **⚠️ NOTE:** Actor Dialogue Events need reverse-engineering. Script Events and standard quests are documented.

| Event Type | Purpose | Example | Status |
|------------|---------|---------|--------|
| **Script Event** | Location-triggered | Entering Neon triggers commentary | Documented |
| **Actor Dialogue Event** | NPC-to-NPC | Companion banter when both present | **NEEDS RE** |
| **(none)** | Standard quest dialogue | Main companion quest | Documented |

---

### A7. PERSONAL CRIME FACTION - ADVANCED SETUP

See **Step 1.11** for basic Personal Crime Faction setup (Crime Tab, Voices Tab).

The `CompanionCrimeResponseScript` checks the "Shared Crime Faction List" to determine who counts as a civilian.

**For Most Companions:**
- Shared list = `FactionSharedCrimeList` (major factions, UC/Freestar civilians)
- Killing anyone in shared list triggers `CivilianKilled` event
- Standard config: Only ignore minor crimes (pickpocket, theft, trespass, piracy)

**Customization by Character Morality:**

**Standard Companion (No Crime):**
- Murder: ☐ Unchecked
- Assault: ☐ Unchecked
- Pickpocket: ☑ Checked
- Smuggling: ☑ Checked
- Stealing: ☑ Checked
- Trespass: ☑ Checked
- Piracy: ☑ Checked

**Criminal/Morally Gray Companion:**
- Murder: ☐ Unchecked (still has limits)
- Assault: ☑ Checked (accepts violence)
- Pickpocket: ☑ Checked
- Smuggling: ☑ Checked
- Stealing: ☑ Checked
- Trespass: ☑ Checked
- Piracy: ☑ Checked

Reference character design document for appropriate morality level.

---

### A8. CUSTOM FACTION FOR FORMER AFFILIATION

For characters with past faction ties (like Cass with Crimson Fleet):

**Problem:** Don't want her treated as current Fleet member (combat issues)

**Solution:** Create `CrimsonFleetFormerFaction`
- NOT the same as `CrimsonFleetFaction`
- Has Fleet in Allied Factions list (for recognition)
- Allows `GetInFaction CrimsonFleetFormerFaction` conditions
- Avoids combat AI treating her as active Fleet

**Example Faction Setup (with Optional Former Affiliation):**
| Faction | Rank | Purpose |
|---------|------|---------|
| `AvailableCompanionFaction` | -1 | Companion system |
| `COM_PersonalCrimeFaction_YourName` | 0 | Crime tracking |
| `CurrentCompanionFaction` | -1 | Active toggle |
| `PlayerAllyFaction` | 0 | Allied with player |
| `FormerAffiliationFaction` | 0 | *Optional* - Former group membership |

The "FormerAffiliationFaction" is useful for characters with past ties to factions (pirates, mercs, etc.) without being treated as current members.

---

## APPLYING THIS GUIDE TO SPECIFIC COMPANIONS

This guide is intentionally generic to serve as a universal reference. To create a specific companion:

1. **Find the companion's design document** (e.g., `Cass_GDD_v1.md`, `Andreja_Breakdown_ARCHIVE.md`)
2. **Extract character-specific information:**
   - Personality, speech patterns, unique behaviors
   - Faction affiliations and morality
   - Affinity reactions and story gates
   - Personal quest and romance content
3. **Apply to each phase:**
   - Replace all `YourName` placeholders with companion name
   - Use character document for dialogue content instead of placeholders
   - Customize perks, factions, and packages based on character role
   - Adapt advanced features (A1-A8) to character backstory

4. **For examples in this guide:**
   - Andreja examples (combat, detection, romance) are generic patterns
   - Adapt wording, timing, and emphasis based on YOUR character's personality
   - Reference `Companion_System_Reference.md` for technical system details

---

*Document Version: 1.2*
*Last Updated: 2026-01-31*
*Status: Generic reference guide - ready for external use*
