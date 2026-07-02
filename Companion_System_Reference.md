# Starfield Companion System - Technical Reference

**FOR EXTERNAL USE** - Comprehensive technical reference for Starfield's companion system architecture and API.

This document reverse-engineers the vanilla companion system using Andreja as an implementation example. The systems, functions, and structures described apply universally to ALL companions.

*v1.2 (2026-07-01): full verification pass against the vanilla Papyrus sources. All function signatures, property names, and call chains now match source. New in this version: the affinity event sending pipeline (Section 7), dismissal side effects (Section 3), the ActiveCompanionChanged CustomEvent (Section 3), and the built-in Andreja jealousy hook (Section 9).*

**Key Sections:**
- **[Script Architecture](#1-script-architecture)** - How components connect
- **[Core Systems](#2-sq_actorrolesscript---foundation)** - Quest and actor scripts
- **[Actor Scripts](#15-actor-attached-scripts)** - Scripts attached to Actor form
- **[API Quick Reference](#14-api-quick-reference)** - Common function calls
- **[Condition Forms](#12-condition-forms)** - Reusable dialogue conditions

---

## TABLE OF CONTENTS

1. [Script Architecture](#1-script-architecture)
2. [SQ_ActorRolesScript - Foundation](#2-sq_actorrolesscript---foundation)
3. [SQ_CompanionsScript - Full Companion System](#3-sq_companionsscript---full-companion-system)
4. [SQ_FollowersScript - Basic Following](#4-sq_followersscript---basic-following)
5. [CompanionActorScript - Actor Attachment](#5-companionactorscript---actor-attachment)
6. [COM_CompanionQuestScript - Personal Quest Manager](#6-com_companionquestscript---personal-quest-manager)
7. [Affinity System](#7-affinity-system)
8. [Anger System](#8-anger-system)
9. [Romance & Commitment System](#9-romance--commitment-system)
10. [Story Gate System](#10-story-gate-system)
11. [Quest Alias System](#11-quest-alias-system)
12. [Condition Forms](#12-condition-forms)
13. [VoiceType System](#13-voicetype-system)
14. [API Quick Reference](#14-api-quick-reference)
15. [Actor-Attached Scripts](#15-actor-attached-scripts)

---

## 1. SCRIPT ARCHITECTURE

### Class Hierarchy

```
Quest (Engine)
  └── SQ_ActorRolesScript (Parent - Role Management)
        ├── SQ_CompanionsScript (Full Companions - affinity, romance, story)
        ├── SQ_FollowersScript (Basic Following - movement, commands)
        └── SQ_CrewScript (Ship Crew - assignment, elite crew)

Actor (Engine)
  └── CompanionActorScript (Companion Actors - ID, affinity functions)
```

### System Relationships

```
┌──────────────────────────────┐
│ COM_Companion_YourName       │  (Personal Quest)
│ └── COM_CompanionQuestScript
└──────────┬───────────────────┘
           │
┌──────────▼───────────────────┐
│ Companion Actor Form         │
│ └── CompanionActorScript
└──────────┬───────────────────┘
           │
      ┌────┼────┬──────────────┐
      ▼    ▼    ▼              ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│SQ_Companions│ │SQ_Followers│ │SQ_Crew│
│(Affinity,   │ │(Movement,   │ │(Ship  │
│ Romance,    │ │Commands)    │ │Crew)  │
│ Story)      │ │             │ │       │
└──────────┘ └──────────┘ └──────────┘
```

### Source File Locations

```
D:\SteamLibrary\steamapps\common\Starfield\Data\Scripts\Source\
├── SQ_ActorRolesScript.psc
├── SQ_CompanionsScript.psc
├── SQ_FollowersScript.psc
├── SQ_CrewScript.psc
├── CompanionActorScript.psc
└── COM_CompanionQuestScript.psc
```

---

## 2. SQ_ACTORROLESSCRIPT - FOUNDATION

**File:** `SQ_ActorRolesScript.psc`
**Extends:** `Quest`
**Purpose:** Parent script providing role management framework

### Core Properties

```papyrus
Group Role_Available
    Alias Property Alias_Available   ; ReferenceAlias OR RefCollectionAlias
    ActorValue Property AV_Available ; Tracks if available for role
    Message Property AvailableMessage
    Message Property NotAvailableMessage
EndGroup

Group Role_Active
    Alias Property Alias_Active      ; ReferenceAlias OR RefCollectionAlias
    ActorValue Property AV_Active    ; Tracks if actively in role
    Message Property ActiveMessage
    Message Property NotActiveMessage
EndGroup
```

### Core Functions

| Function | Parameters | Returns | Purpose |
|----------|------------|---------|---------|
| `SetRoleAvailable` | Actor, DisplayMsg | void | Makes actor eligible |
| `SetRoleUnavailable` | Actor, DisplayMsg | void | Removes eligibility |
| `SetRoleActive` | Actor, DisplayMsg, AlsoSetAvailable=**true**, MessageFloat1=0.0, MessageFloat2=0.0 | void | Activates actor (auto-calls SetRoleAvailable and, at the end, EvaluatePackage) |
| `SetRoleInactive` | Actor, DisplayMsg, AlsoSetUnavailable=**false** | void | Deactivates actor (stays *available* by default — re-recruitable) |
| `IsRoleActive` | Actor | bool | Check if active |
| `IsRoleAvailable` | Actor | bool | Check if available |
| `GetActiveActor` | none | Actor | Get first/only active |
| `GetActiveActors` | none | Actor[] | Get all active |
| `GetAvailableActor` | none | Actor | Get first/only available |
| `GetAvailableActors` | none | Actor[] | Get all available |

### Alias Types

- **ReferenceAlias** = Only ONE actor can be in role (e.g., Active Companion)
- **RefCollectionAlias** = MULTIPLE actors can be in role (e.g., Available Companions)

### SQ_PreventRecalc System

```papyrus
Function _UpdateActorValue(Actor ActorToUpdate, ActorValue ActorValueToUpdate, bool TurnOn = true)
    ; Prevent AV reset when player first sees actor
    SQ_PreventRecalc.AddRef(ActorToUpdate)

    int value = 0
    if TurnOn
        value = 1
    endif
    ActorToUpdate.SetValue(ActorValueToUpdate, value)
EndFunction
```

*(Reminder: Papyrus has no ternary operator — all conditionals are if/endif blocks.)*

**Critical:** Prevents companion stats from being reset when first loaded!

---

## 3. SQ_COMPANIONSSCRIPT - FULL COMPANION SYSTEM

**File:** `SQ_CompanionsScript.psc`
**Extends:** `SQ_ActorRolesScript`
**Purpose:** Complete companion management with affinity, romance, anger, story

### Key Properties

```papyrus
; (abridged — group and property names match source)
Group Affinity
    ActorValue Property COM_Affinity                        ; Current affinity value
    ActorValue Property COM_AffinityLevel                   ; 0-3 level
    GlobalVariable Property COM_AffinityLevel_0_Neutral     ; level enum globals:
    GlobalVariable Property COM_AffinityLevel_1_Friendship  ;   read the level by comparing
    GlobalVariable Property COM_AffinityLevel_2_Affection   ;   the AV against these
    GlobalVariable Property COM_AffinityLevel_3_Commitment
    GlobalVariable[] Property COM_AffinityLevel_EnumGlobals
    AffinityLevelDatum[] Property AffinityLevelData
EndGroup

Group Anger
    ActorValue Property COM_AngerLevel
    ActorValue Property COM_AngerSceneCompleted
    ActorValue Property COM_AngerReason
    GlobalVariable Property COM_AngerLevel_0_NotAngry       ; also _1_Annoyed, _2_Angry, _3_Furious
    AngerLevelDatum[] Property AngerLevelData
EndGroup

Group Romance
    Keyword Property COM_SleepTopic_PlayerWakesUp
    Keyword Property COM_SleepTopic_WakeUpInPlayersBed
EndGroup
```

### _CustomSetRoleActive Override

When `SetRoleActive()` is called, this chain executes:

```papyrus
Function _CustomSetRoleActive(Actor ActorToUpdate, Actor PriorActiveActor)
    ; 1. Deactivate previous companion
    if PriorActiveActor && PriorActiveActor != ActorToUpdate
        SetRoleInactive(PriorActiveActor)
    endif

    ; 2. Remove elite crew (they say their unassigned line)
    SQ_Crew.SetEliteCrewInactive(ActiveEliteCrew.GetActorReference(), sayUnassignedLine = true)

    ; 3. ALSO set as FOLLOWER (enables following behavior)
    SQ_Followers.SetRoleActive(ActorToUpdate, DisplayMessageIfChanged = false, AlsoSetAvailable = true)

    ; 4. Make available as crew
    SQ_Crew.SetRoleAvailable(ActorToUpdate)

    ; 5. Fire companion changed event — ONLY if the companion actually changed
    if PriorActiveActor != ActorToUpdate
        _SendActiveCompanionChanged(ActorToUpdate, PriorActiveActor)
    endif
EndFunction
```

**Key Insight:** `SQ_Companions.SetRoleActive()` internally calls `SQ_Followers.SetRoleActive()`!

### Dismissal Side Effects (_CustomSetRoleInactive)

Dismissing is not just "stop following." `_CustomSetRoleInactive` also:
- Calls `SQ_Followers.SetRoleInactive()`
- Fires `_SendActiveCompanionChanged(None, DismissedActor)`
- Stops the companion's combat
- Fake-assigns then unassigns the companion as crew, so they walk back to the ship

**Important default:** `SetRoleInactive()` has `AlsoSetUnavailable = false` by default — a dismissed companion **stays available** and can be re-recruited through dialogue.

### ActiveCompanionChanged CustomEvent

`SQ_CompanionsScript` fires a `ActiveCompanionChanged` CustomEvent whenever the active companion changes, with args struct `{NewActiveCompanion, OldActiveCompanion}` (helper: `GetActiveCompanionChangedArgsStruct()`). External scripts can `RegisterForCustomEvent(SQ_Companions, "ActiveCompanionChanged")` — the clean hook for reacting to companion swaps (including cross-mod companion detection).

### Companion Check Functions

To check a companion's affinity level, read the `COM_AffinityLevel` actor value directly (set via `SetAffinityLevel()`, read via `GetValueEnumGlobal(COM_AffinityLevel, COM_AffinityLevel_EnumGlobals)` or a plain `GetValue()` compare) — the script provides no per-level convenience wrappers. The check functions it does provide:

```papyrus
bool Function IsCompanion(Actor ActorToCheck, bool IncludeAvailableCompanions = true)

bool Function IsCompanionRomantic(CompanionActorScript Companion)
    return Companion.GetValue(SQ_Companions.COM_IsRomantic) >= 1
EndFunction

bool Function IsCompanionLockedIn()
    ; true when COM_PQ_LockedInCompanion > -1 (a quest has locked the companion)
EndFunction
```

Note `IsCompanionRomantic` takes a `CompanionActorScript`, not a plain `Actor`.

---

## 4. SQ_FOLLOWERSSCRIPT - BASIC FOLLOWING

**File:** `SQ_FollowersScript.psc`
**Extends:** `SQ_ActorRolesScript`
**Purpose:** Physical following behavior, movement commands

### Key Functions

| Function | Purpose |
|----------|---------|
| `CommandFollow(Actor)` | Tell follower to follow player |
| `CommandWait(Actor)` | Tell follower to wait in place |
| `TeleportWaitingFollowersToShip(Location akNewLoc = None)` | Move waiting followers to the ship |
| `TeleportFollowers(ObjectReference DestinationRef, Actor[] SpecificFollowersToTeleport = None, ...)` | General follower teleport |
| `SetIdleChatterTimes(Actor, IsFollower)` | Configure talk frequency |
| `GetScript() global` | `Game.GetFormFromFile(0x0000D773, "Starfield.esm")` — lets external scripts grab SQ_Followers without a property |

### _CustomSetRoleActive Override

```papyrus
; (paraphrased — source declares the AVs locally, e.g.
;  ActorValue aggressionAV = Game.GetAggressionAV())
Function _CustomSetRoleActive(Actor ActorToUpdate, Actor PriorActiveActor)
    SetGroupFormationFactionData(ActorToUpdate)
    ActorToUpdate.SetPlayerTeammate(true, abCanDoFavor = false)
    SetIdleChatterTimes(ActorToUpdate, IsFollower = true)
    ActorToUpdate.SetNotShowOnStealthMeter(true)
    ActorToUpdate.SetValue(Cached_PreFollowerAggression, aggression)  ; cache current aggression
    ActorToUpdate.SetValue(aggressionAV, 0)                           ; then zero it while following
    CommandFollow(ActorToUpdate)
EndFunction
```

---

## 5. COMPANIONACTORSCRIPT - ACTOR ATTACHMENT

**File:** `CompanionActorScript.psc`
**Extends:** `Actor`
**Purpose:** Script attached directly to companion actor forms

### Required Properties

```papyrus
Group Autofill
    SQ_CompanionsScript Property SQ_Companions Mandatory Const Auto
    ActorValue Property COM_Affinity Mandatory Const Auto
    Keyword Property COM_PreventStoryGateScenes Mandatory Const Auto
EndGroup

Group Properties
    COM_CompanionQuestScript Property COM_CompanionQuest Mandatory Const Auto
    GlobalVariable Property COM_CompanionID Mandatory Const Auto
    Activator Property COM_PQ_TxtReplace_QuestName Mandatory Const Auto
EndGroup
```

### Key Functions

```papyrus
GlobalVariable Function GetCompanionIDGlobal()
    return COM_CompanionID
EndFunction

float Function GetCompanionIDValue()
    return GetCompanionIDGlobal().GetValue()
EndFunction

bool Function IsRomantic()
    return SQ_Companions.IsCompanionRomantic(self)
EndFunction

Function AddAffinity(int AmountToAdd)
    AddPassiveAffinity(AmountToAdd as Float)
    COM_CompanionQuest.CheckAndSetWantsToTalk()
EndFunction

; Story gate helpers (useful for testing and quest fragments):
Function AllowStoryGatesAndRestartTimer()   ; removes COM_PreventStoryGateScenes keyword + restarts gate timer
Function RestartStoryGateTimer(int nextStoryGate = 1)
int Function GetCompanionIDValueInt()
bool Function HasGreaterAffinityThanTestedCompanion()
```

### OnDeath Event

```papyrus
Event OnDeath(ObjectReference akKiller)
    SQ_Companions.SetRoleInactive(self)
    SQ_Companions.SetRoleUnavailable(self)
EndEvent
```

---

## 6. COM_COMPANIONQUESTSCRIPT - PERSONAL QUEST MANAGER

**File:** `COM_CompanionQuestScript.psc`
**Extends:** `Quest`
**Attached To:** Each `COM_Companion_<Name>` quest

### Key Properties

```papyrus
Group Aliases
    ReferenceAlias Property Alias_Companion Mandatory Const Auto
    ReferenceAlias Property Alias_PlayerShipCrewMarker Mandatory Const Auto
    LocationAlias Property Alias_PlayerShipInteriorLocation Mandatory Const Auto
EndGroup

Group StoryGates
    StoryGateDatum[] Property StoryGateData Mandatory Const Auto
    ActorValue Property COM_StarbornSaveActorValue_MaxStoryGate Mandatory Const Auto
    ; Filter: COM_StarbornSaveActorValue_MaxStoryGate_*
    ; IMPORTANT: Must also be in formlist StarbornSaveActorValues
EndGroup

Group Quests
    Quest Property PersonalQuest Mandatory Const Auto
    Quest Property CommitmentQuest Mandatory Const Auto
EndGroup

Group Anger
    GlobalVariable[] Property PrioritizedAngerReasons Mandatory Const Auto
    ; Filter: COM_AngerReason_*
    ; Lower indexes = higher priority
    ; New anger reasons ignored if current reason has lower index
    ; Typically includes only serious offenses like murder
EndGroup

Group WantsToTalk
    Scene Property WantsToTalkAnnouncementScene Mandatory Const Auto
    ConditionForm Property WantsToTalkConditionForm Mandatory Const Auto
    int Property WantsToTalkObjective = 10 Const Auto
EndGroup

Group Autofill
    ; NOTE: includes Companion_Andreja and COM_Event_PlayerBecomesRomantic_AndrejaJealous —
    ; EVERY quest using this script must fill the Andreja-related properties (they drive
    ; the vanilla jealousy check in MakeRomantic; see Section 9)
EndGroup

Group AdditionalData
    Perk Property CompanionCheckPerk Mandatory Const Auto
    ; per-companion perk used for dialogue interjections
EndGroup

Group Flirting
    GlobalVariable Property COM_FlirtCooldownDays Mandatory Const Auto
    ActorValue Property COM_FlirtCount Mandatory Const Auto
    ActorValue Property COM_FlirtCooldownExpiry Mandatory Const Auto
    ActorValue Property COM_FlirtChoice Mandatory Const Auto
    int Property FlirtChoiceMax = 2 Const Auto
EndGroup
```

### Event Registrations

```papyrus
Event OnQuestInit()
    CompanionRef = Alias_Companion.GetActorRef() as CompanionActorScript
    RegisterForRemoteEvent(CompanionRef, "OnAffinityEvent")
    RegisterForRemoteEvent(CompanionRef, "OnLocationChange")
    RegisterForRemoteEvent(Game.GetPlayer(), "OnLocationChange")
    RegisterForCustomEvent(SQ_Companions, "ActiveCompanionChanged")
EndEvent
```

### Scene Callback Functions

| Function | Called When |
|----------|-------------|
| `PickupSceneEnded()` | Player recruits companion |
| `DismissSceneEnded()` | Player dismisses companion |
| `WaitSceneEnded()` | Player tells companion to wait |
| `FollowSceneEnded()` | Player tells companion to follow |
| `GiveItemSceneEnded()` | Player gives item |
| `FlirtSceneEnded()` | Flirt conversation ends |
| `StoryGateSceneCompleted()` | Story gate conversation ends |
| `AngerSceneCompleted()` | Anger confrontation ends |
| `TalkAboutQuestEventSceneEnded(ActorValue)` | Quest-event conversation ends |
| `RomanceSceneEndedRomantic()` | Romance scene ends with player choosing romance |

---

## 7. AFFINITY SYSTEM

### Affinity Levels

| Level | Value | Global Variable | Meaning |
|-------|-------|-----------------|---------|
| Neutral | 0 | `COM_AffinityLevel_0_Neutral` | Just met |
| Friendship | 1 | `COM_AffinityLevel_1_Friendship` | Getting along |
| Affection | 2 | `COM_AffinityLevel_2_Affection` | Deep bond |
| Commitment | 3 | `COM_AffinityLevel_3_Commitment` | Romance possible |

### Affinity Event Reactions

| Reaction | Constant | Points |
|----------|----------|--------|
| LOVES | `COM_EventReaction_Loves` | +50 |
| LIKES | `COM_EventReaction_Likes` | +25 |
| INDIFFERENT | `COM_EventReaction_Indifferent` | 0 |
| DISLIKES | `COM_EventReaction_Dislikes` | -15 |
| HATES | `COM_EventReaction_Hates` | -35 |

*(Point values are GlobalVariable data, confirmed against the CK data dump rather than script source.)*

### Affinity Event Sending Pipeline (CompanionAffinityEventsScript)

The receiving side (below and Section 15) is only half the system. All `COM_Event_Action_*` events are **sent** by one vanilla quest script: `CompanionAffinityEventsScript.psc`. `AffinityEvent` itself is a native Form type with just two functions — `Send(ObjectReference akTarget = None)` and `Reset()`.

**The full chain:**
```
CompanionAffinityEventsScript (sender quest — watches player actions)
    └─► AffinityEvent.Send()
          └─► CompanionAffinityScript.OnAffinityEvent (on each qualifying actor)
                ├─► anger check (AngerEventData)
                └─► SQ_Companions.SQ_Traits.HandleAffinityEvent()  (points applied)
```

**Send-gating rules (why lines don't always play):**
- **One global cooldown for ALL action events:** `COM_ActionEventScriptFilter_CoolDownMinutes` (minutes × 60). After any action event fires, all others are suppressed until it expires. A FormList (`COM_IgnoreAffinityEventCooldownOnChangeLocation_Locations`) lets specific locations bypass it for Arrival.
- **Events are dropped, not queued,** when the script is cooling down, the player is in dialogue, the companion is in a scene, or player/companion is within 20m of a configured "important scene" (`ImportantSceneData`).
- **Steal/Pickpocket require line-of-sight:** `CompanionRef.HasDetectionLOS(PlayerRef)` — the companion must see the theft.
- **Gravity thresholds:** high > 1.5g, low < 0.5g, polled after location changes. ⚠ The ZeroG branch is **unreachable** (`< 0.5` matches GravityLow before `<= 0` is tested) — quests must `Send()` `COM_Event_Action_ZeroG` directly.
- ⚠ **`COM_Event_Action_UseWorkbench` is dead:** its handler is commented out in vanilla ("no companion lines were written for this", bug GEN-432521).

**Modding implication:** to fire a vanilla action event from your own quest, just call `TheEvent.Send()`. To add reactions to existing events, no script is needed — reactions are data on the AffinityEvent form.

### Level Progression Functions

```papyrus
Function StartPersonalQuest()
    StoryGateSceneCompleted()   ; increments the gate counter + starts next timer FIRST
    SetAffinityLevel(COM_AffinityLevel_1_Friendship)
    PersonalQuest.Start()
    CompanionRef.SetValue(COM_PersonalQuest_Started, 1)
    ; (ends by showing a text-replaced "advancing quest" warning message)
EndFunction

Function FinishedPersonalQuest()
    SetAffinityLevel(COM_AffinityLevel_2_Affection)
    CompanionRef.SetValue(COM_PersonalQuest_Finished, 1)
EndFunction
```

---

## 8. ANGER SYSTEM

### Anger Levels

| Level | Global Variable | Effect |
|-------|-----------------|--------|
| 0 | `COM_AngerLevel_0_NotAngry` | Normal |
| 1 | `COM_AngerLevel_1_Annoyed` | Dialogue changes |
| 2 | `COM_AngerLevel_2_Angry` | May leave, confrontation possible |
| 3 | `COM_AngerLevel_3_Furious` | Will leave, relationship damaged |

### Anger Functions

```papyrus
Function SetAngerLevel(GlobalVariable AngerLevelToSet, GlobalVariable AngerReason)
    SQ_Companions.SetAngerLevel(CompanionRef, AngerLevelToSet, AngerReason)
    CheckAndSetWantsToTalk()
EndFunction

Function MakeNotAngry()
    SetAngerLevel(SQ_Companions.COM_AngerLevel_0_NotAngry,
                  SQ_Companions.COM_AngerReason_Common_0_NotAngry)
EndFunction

Function StartAngerCoolDownTimer()
    float coolDownTimerDur = SQ_Companions.GetAngerCoolDownTimerDuration(CompanionRef)
    if coolDownTimerDur > -1
        StartTimerGameTime(coolDownTimerDur, GameTimerID_AngerCoolDown)
        CompanionRef.SetValue(COM_AngerCoolDownTimerExpired, 0)
    else
        CancelTimerGameTime(GameTimerID_AngerCoolDown)  ; -1 duration = no cooldown timer
    endif
    CheckAndSetWantsToTalk()
EndFunction
```

### Second Chances

```papyrus
Function AwardSecondChance()
    float newVal = CompanionRef.GetValue(COM_AngerSecondChances) + 1
    CompanionRef.SetValue(COM_AngerSecondChances, newVal)
EndFunction

Function ReedemSecondChance()
    float newVal = CompanionRef.GetValue(COM_AngerSecondChances) - 1
    CompanionRef.SetValue(COM_AngerSecondChances, newVal)
    _AngerSceneCancelAnger()
EndFunction
```

### Prioritized Anger Reasons

The `PrioritizedAngerReasons` property controls which anger reasons cannot be overridden by newer, less serious offenses.

**How It Works:**
- Array of GlobalVariables (filter: `COM_AngerReason_*`)
- Lower array index = higher priority
- If companion is already angry for a reason in this list, new anger reasons with higher index are ignored
- Prevents minor offenses from overriding serious ones like murder

**Example Configuration:**
```
Element 0: COM_AngerReason_Common_1_Murder (highest priority)
Element 1: COM_AngerReason_Common_3_BreakCommitment
Element 2: COM_AngerReason_YourName_SpecificBetrayal
```

**Usage:** Typically only includes 1-3 serious offenses that should never be superseded by lesser anger triggers.

---

## 9. ROMANCE & COMMITMENT SYSTEM

### Romance Functions

```papyrus
Function MakeRomantic()
    CompanionRef.SetValue(COM_IsRomantic, 1)
    if CompanionRef.GetValue(COM_HasBeenRomantic) < 1
        CompanionRef.SetValue(COM_HasBeenRomantic, 1)   ; first-time flag, set once
    endif
    PossiblyMakeAndrejaJealous()
EndFunction

Function MakeNotRomantic()
    CompanionRef.SetValue(COM_IsRomantic, 0)
    CommitmentDesired(false)
EndFunction
```

> ⚠ **The Andreja jealousy hook is baked into the shared script.** `PossiblyMakeAndrejaJealous()` fires `COM_Event_PlayerBecomesRomantic_AndrejaJealous.Send(romanticCompanion)` whenever the player romances anyone while Andreja is romantic. This means **romancing a CUSTOM companion will anger a romanced Andreja** automatically — no extra work needed (or possible to skip, short of modifying the event).

### Commitment Functions

```papyrus
Function CommitmentDesired(bool Desired = true)
    if Desired
        CompanionRef.setValue(COM_CommitmentDesired, 1)
    else
        CompanionRef.setValue(COM_CommitmentDesired, 0)
    endif
    CompanionRef.SetValue(COM_CommitmentPossible, 1)
    ; If the current story gate timer already expired ("I need some time" case),
    ; the source also resets that AV and starts a short 600s delay timer:
    ;   CompanionRef.SetValue(COM_CurrentStoryGateTimerExpired, 0)
    ;   StartTimer(600, TimerID_StoryGate)
EndFunction

Function StartCommitmentQuest()
    StoryGateSceneCompleted()
    CompanionRef.SetValue(COM_CommitmentQuest_Started, 1)
    CommitmentQuest.Start()
EndFunction

Function MakeCommitted()
    SetAffinityLevel(COM_AffinityLevel_3_Commitment)
    CompanionRef.SetValue(COM_IsCommitted, 1)
EndFunction

Function BreakCommitment()
    MakeNotCommitted(Permanent = true)
    MakeNotRomantic()
    SetAngerLevel(SQ_Companions.COM_AngerLevel_2_Angry,
                  SQ_Companions.COM_AngerReason_Common_3_BreakCommitment)
    StartAngerCoolDownTimer()
EndFunction
```

---

## 10. STORY GATE SYSTEM

### StoryGateDatum Struct

```papyrus
Struct StoryGateDatum
    int GateNumber
    GlobalVariable TimerDuration  ; in seconds
EndStruct
```

**How to Create in Creation Kit:**

In your `COM_Companion_YourName` quest's script properties:

1. Find the `StoryGateData` property (array of StoryGateDatum)
2. Set array size (e.g., 6 for 6 story gates)
3. For each element:
   - `GateNumber` = 1, 2, 3, etc.
   - `TimerDuration` = GlobalVariable with duration in seconds

**Example Configuration:**
```
Element 0: GateNumber=1, TimerDuration=COM_StoryGate_TimerDuration_01_Standard
Element 1: GateNumber=2, TimerDuration=COM_StoryGate_TimerDuration_02_Standard
Element 2: GateNumber=3, TimerDuration=COM_StoryGate_TimerDuration_03_Standard
...
```

**Note:** The source struct comment says to filter for `COM_StoryGate_TimerDuration` — vanilla ships `COM_StoryGate_TimerDuration_01` through `_08_Standard`. Reuse those (as the Cass quest does) or create your own GlobalVariables with custom durations in seconds.

### Story Gate Flow

```
1. Player recruits companion
   └─► PickedUpAsCompanion() starts first timer

2. Timer runs while companion is active
   └─► Paused when dismissed, resumed when re-recruited

3. Timer expires
   └─► COM_CurrentStoryGateTimerExpired = 1
   └─► CheckAndSetWantsToTalk() fires

4. Player talks to companion
   └─► Story Gate scene plays
   └─► StoryGateSceneCompleted() increments counter
   └─► Next timer starts
```

### Key Functions

```papyrus
Function StoryGateSceneCompleted(bool IncrementAV = true)
    float currentAV = CompanionRef.GetValue(COM_StoryGatesCompleted)
    float newAV = currentAV
    if IncrementAV
        newAV = currentAV + 1
    endif
    CompanionRef.SetValue(COM_StoryGatesCompleted, newAV)

    ; NG+ carry-over: the max gate reached is written to the PLAYER as
    ; COM_StarbornSaveActorValue_MaxStoryGate (Math.Max with current value) —
    ; this is exactly why that AV must be in the StarbornSaveActorValues formlist
    ; (see Section 6, StoryGates group)

    int nextGateNumber = (newAV + 1) as int
    StartStoryGateTimer(nextGateNumber)
    CheckAndSetWantsToTalk()
EndFunction

Function StartStoryGateTimer(int NextGateNumber)
    StoryGateDatum nextDatum = GetStoryGateDatum(NextGateNumber)
    if nextDatum
        CompanionRef.setValue(COM_CurrentStoryGateTimerExpired, 0)
        StartTimer(nextDatum.TimerDuration.GetValue(), TimerID_StoryGate)
    endif
EndFunction
```

---

## 11. QUEST ALIAS SYSTEM

### External Alias References

Quests can "borrow" aliases from other quests:

```
COM_Companion_Andreja.ActiveCompanion
    └─► Links to ─► SQ_Companions.ActiveCompanion
```

### External Alias Setup

```
Alias Name: ActiveCompanion
Fill Type: ● External Alias Ref
           ☑ (Linked)
           Quest: SQ_Companions
           Alias: ActiveCompanion
Flags: ☑ Optional
```

### SQ_Companions Master Aliases

| Alias Name | Type | Purpose |
|------------|------|---------|
| `ActiveCompanion` | Ref | Currently active companion |
| `availableCompanions` | RefCollection | All available companions |
| `SleepCompanionRomantic` | Ref | Romantic partner for sleep |
| `SleepCompanion` | Ref | Companion sleeping nearby |
| `MessageTextReplaceActor` | Ref | For message text replacement |
| `ActiveEliteCrew` | Ref | Active elite crew member |
| `PlayerShip` | Ref | Player's ship |
| `CommitmentGifts` | RefCollection | Wedding gifts |

---

## 12. CONDITION FORMS

Starfield introduced **Condition Forms** - reusable condition packages.

### Common Condition Forms

| ConditionForm | Purpose |
|---------------|---------|
| `COM_CND_DIAL_Hello_Angry` | Angry hello check |
| `COM_CND_DIAL_Hello_Friendship` | Friendship hello check |
| `COM_CND_DIAL_Hello_Affection` | Affection hello check |
| `COM_CND_DIAL_Hello_Committed` | Committed hello check |
| `COM_CND_DIAL_Goodbye_*` | Goodbye variants |
| `COM_CND_DIAL_Dismiss_Positive` | Positive dismissal |
| `COM_CND_DIAL_Dismiss_Negative` | Negative dismissal |
| `COM_CND_DIAL_WaitingForInput_*` | WFI variants |

### Usage in Conditions

```
Target | Function | Info | Comp | Value
S | IsTrueForConditionForm | 'COM_CND_DIAL_Hello_Angry' | == | 1.0000
```

---

## 13. VOICETYPE SYSTEM

### Default_CompanionVoicesList

FormList containing all companion VoiceTypes:

| Editor ID | Form Type |
|-----------|-----------|
| `NPCFAndreja` | VoiceType |
| `NPCMBarrett` | VoiceType |
| `NPCMSamCoe` | VoiceType |
| `NPCFSarahMorgan` | VoiceType |

### VoiceType Conditions

Used to filter companion-specific dialogue:

```
Target | Function | Info | Comp | Value
S | GetIsVoiceType | VoiceType: 'NPCFAndreja' | == | 1.0000
```

---

## 14. API QUICK REFERENCE

### Make Companion Available

```papyrus
(SQ_Companions as SQ_CompanionsScript).SetRoleAvailable(CompanionREF)
```

### Make Companion Active

```papyrus
If COM_PQ_LockedInCompanion.GetValueInt() > -1
    kmyquest.ShowHelpMessage("CompanionBlocked")
Else
    (SQ_Companions as SQ_CompanionsScript).SetRoleActive(CompanionREF)
    CompanionREF.EvaluatePackage()
EndIf
```

### Dismiss Companion

```papyrus
(SQ_Companions as SQ_CompanionsScript).SetRoleInactive(CompanionREF)
```

### Check If Active

```papyrus
bool isActive = (SQ_Companions as SQ_CompanionsScript).IsRoleActive(CompanionREF)
```

### Check Affinity Level

```papyrus
int level = CompanionREF.GetValue(COM_AffinityLevel) as int
; 0 = Neutral, 1 = Friendship, 2 = Affection, 3 = Commitment
```

### Check If Romantic

```papyrus
bool romantic = (CompanionREF as CompanionActorScript).IsRomantic()
```

### Add Affinity

```papyrus
(CompanionREF as CompanionActorScript).AddAffinity(25)
```

### Full SetRoleActive Call Chain

```
1. YOUR SCRIPT calls SQ_Companions.SetRoleActive(CompanionREF)
   │
   ├─► 2. SetRoleAvailable() called AUTOMATICALLY (AlsoSetAvailable defaults true)
   │      └─► PotentialCrewFaction, SQ_Crew.SetRoleAvailable(), achievement check
   │
   ├─► 3. _UpdateAlias() - Adds to Alias_Active
   │
   ├─► 4. _UpdateActorValue() - Sets AV_Active = 1
   │      └─► SQ_PreventRecalc.AddRef() to protect stats
   │
   ├─► 5. _CustomSetRoleActive()
   │      ├─► SetRoleInactive(PriorCompanion) (if different)
   │      ├─► SQ_Crew.SetEliteCrewInactive(..., sayUnassignedLine = true)
   │      ├─► SQ_Followers.SetRoleActive() ─► CommandFollow()
   │      ├─► SQ_Crew.SetRoleAvailable()
   │      └─► _SendActiveCompanionChanged() (only if companion changed)
   │
   └─► 6. SetRoleActive() itself calls CompanionREF.EvaluatePackage() LAST
```

**Note:** the separate `SetRoleAvailable()` and `EvaluatePackage()` calls seen in vanilla stage fragments (and this doc's snippets above) are **harmless but redundant** — `SetRoleActive()` already does both. Follow the vanilla pattern for consistency, but know it's belt-and-suspenders.

---

## 15. ACTOR-ATTACHED SCRIPTS

These scripts are attached directly to the companion Actor form and handle specific behaviors.

### CompanionAffinityScript

**File:** `CompanionAffinityScript.psc`
**Extends:** `Actor`
**Purpose:** Handles affinity event reactions and anger system

#### Anger Event Struct

```papyrus
Struct AngerEvent
    AffinityEvent EventCausingAnger
    GlobalVariable AngerLevel      ; filter: COM_AngerLevel
    GlobalVariable AngerReason     ; filter: COM_AngerReason
EndStruct
```

#### Anger Levels (GlobalVariable)

| Level | Editor ID | Effect |
|-------|-----------|--------|
| 0 | `COM_AngerLevel_0_NotAngry` | Normal state |
| 1 | `COM_AngerLevel_1_Annoyed` | Mild annoyance, dialogue changes |
| 2 | `COM_AngerLevel_2_Angry` | May leave, confrontation possible |
| 3 | `COM_AngerLevel_3_Furious` | Will leave, relationship damaged |

#### Anger Reasons (GlobalVariable)

| Value | Editor ID | Trigger |
|-------|-----------|---------|
| 0 | `COM_AngerReason_Common_0_NotAngry` | Default/reset state |
| 1 | `COM_AngerReason_Common_1_Murder` | Player killed civilian |
| 2 | `COM_AngerReason_Common_2_Jealousy` | Romance betrayal (Andreja-specific; CK data — not named in script source) |
| 3 | `COM_AngerReason_Common_3_BreakCommitment` | Player broke commitment |

#### Key Properties

| Property | Type | Notes |
|----------|------|-------|
| `SQ_Companions` | SQ_CompanionsScript | Autofill |
| `AngerEventData` | AngerEvent[] | Array of events that trigger anger |

#### Key Event

```papyrus
Event OnAffinityEvent(AffinityEvent akEvent, ActorValue akAV,
                      GlobalVariable akReactionValue, ObjectReference akTarget)
    ; 1. Checks if event triggers anger
    ; 2. Calls COM_CompanionQuest.SetAngerLevel() if angry
    ; 3. Passes to SQ_Traits.HandleAffinityEvent() for processing
EndEvent
```

> This is the **receiving** end. For where these events come from and why they sometimes don't fire (global cooldown, drop rules, line-of-sight), see **Section 7: Affinity Event Sending Pipeline**.

#### Common AffinityEvents for AngerEventData

**`COM_Event_Action_CivilianCombat_Common`**
- Description: Player in combat with civilian/innocent
- Triggers auto-dismissal
- Vanilla companions: All DISLIKE (except Sophia Grace = Indifferent)

**`COM_Event_Action_CivilianKilled_Common`**
- Description: Player killed a civilian/innocent
- Triggers auto-dismissal and anger state
- Vanilla companions: All HATE

#### Vanilla Example: Andreja's AngerEventData

| # | EventCausingAnger | AngerLevel | AngerReason |
|---|-------------------|------------|-------------|
| 0 | `COM_Event_Action_CivilianCombat_Common` | `COM_AngerLevel_2_Angry` | None |
| 1 | `COM_Event_Action_CivilianKilled_Common` | `COM_AngerLevel_2_Angry` | `COM_AngerReason_Common_1_Murder` |
| 2 | `COM_Event_PlayerBecomesRomantic_AndrejaJealous` | `COM_AngerLevel_2_Angry` | `COM_AngerReason_Common_2_Jealousy` |

> **Note:** The jealousy event (#2) is **Andreja-specific**. Other vanilla companions (Barrett, Sam, Sarah) do NOT have jealousy events in their AngerEventData. This was a unique design choice for Andreja's character.

#### Implementation Notes

- **Morally grey companions** can use lower anger levels (e.g., `Annoyed` instead of `Angry`) for civilian events
- **Jealousy system** is optional - only implement if fits character (requires companion-specific AffinityEvent like `COM_Event_PlayerBecomesRomantic_YourNameJealous`)
- Anger levels affect dialogue and whether companion leaves - Level 2+ may trigger auto-dismissal

---

### CompanionCrimeResponseScript

**File:** `CompanionCrimeResponseScript.psc`  
**Extends:** `Actor`  
**Purpose:** Handles companion reaction to player crimes against civilians

#### Key Properties

| Property | Type | Filter/Notes |
|----------|------|--------------|
| `SQ_Companions` | SQ_CompanionsScript | Autofill |
| `CurrentCompanionFaction` | Faction | Autofill |
| `COM_AngerReason_Common_1_Murder` | GlobalVariable | Autofill |
| `COM_NotCivilian` | Keyword | Autofill |
| `AffinityEventOnCombat` | AffinityEvent | **Filter: `COM_Event_Action_CivilianCombat`** |
| `AffinityEventOnKill` | AffinityEvent | **Filter: `COM_Event_Action_CivilianKilled`** |
| `IgnoreSharedCrimeForAnyoneInTheseFactions` | Faction[] | Factions to ignore for crime checks |

#### Critical Mechanic

**Personal Crime Faction Required!**

> All actors with this script MUST have their own 'personal crime faction' that has a shared crime faction list of factions they consider 'civilians'

**Flow:**
1. Registers for player assault/murder events AND `OnCombatStateChanged` (sweeps all combatants, not just direct assaults)
2. Checks if victim is in companion's "shared crime faction list"
3. If yes → `CivilianCombat()` or `CivilianKilled()` → Sends affinity event → anger state set; dismissal (if any) comes from the **anger system**, not a direct `AutoDismiss()` call

#### Key Functions

| Function | Purpose |
|----------|---------|
| `IsValidCrimeVictim()` | Checks if actor counts as civilian to this companion |
| `CivilianCombat()` | Pacifies companion, sends event, may dismiss |
| `CivilianKilled()` | Same but for kills |
| `AutoDismiss()` | Removes companion if not locked in |

**Customization:** Adjust Personal Crime Faction's "Ignore Crimes Against Non-Members" checkboxes based on companion morality.

---

### COM_CREW_GiveItemActorScript

**File:** `COM_CREW_GiveItemActorScript.psc`  
**Extends:** `Actor`  
**Purpose:** Companions/crew periodically give player items (gifts)

#### Key Properties

| Property | Type | Notes |
|----------|------|-------|
| `SQ_Crew` | SQ_COM_CREW_GiveItemQuestScript | Autofill |
| `COM_CREW_GiveItem_HasItems` | ActorValue | Autofill |
| `COM_CREW_GiveItemAvailableCooldownDays` | GlobalVariable | Days between gifts |
| `COM_CREW_GiveItemAvailableDay` | ActorValue | Autofill |
| `COM_CREW_GiveItemReminder` | Keyword | Dialogue subtype for reminder |
| `COM_CREW_CND_GiveItemAvailable` | ConditionForm | Autofill |
| `GiveItemList` | LeveledItem | **Filter: `GiveItemList`** - Items to give |

#### Mechanic

```
1. First OnLocationChange only STARTS a game-time cooldown timer
   (DaysAsHours(CooldownDays)) - no gift check yet
2. After the timer fires, location changes check the cooldown
3. If conditions met, sets HasItems = 1
4. Says reminder line via SayCustom()
5. GiveItems() adds from leveled list to player
```

**Implementation:** Create a `LeveledItem` list with character-appropriate gifts (ammo, alcohol, ship parts, credits, etc.).

---

### CompanionDebugScript

**File:** `CompanionDebugScript.psc`  
**Extends:** `Actor`  
**Purpose:** Console debug commands for testing

#### Key Functions (callable from console)

| Function | Purpose |
|----------|---------|
| `DebugMakeAvailableCompanion()` | Sets available, moves to player |
| `DebugMakeActiveCompanion()` | Sets quest stages, makes active |
| `DebugMakeInactiveCompanion()` | Dismisses companion |
| `DebugSetAffinityLevel(GlobalVariable)` | Force affinity level |
| `DebugModAffinity(float)` | Add/remove affinity points |
| `DebugSetStoryGateTimerComplete()` | Skip story gate timer |
| `DebugSetAngerLevel(GlobalVariable, GlobalVariable)` | Force anger state |
| `DebugExpireAngerCoolDownTimer()` | Clear anger cooldown |
| `DebugAwardSecondChance()` | Give an anger-system second chance |
| `DebugMakeRomantic()` | Force romance state |
| `DebugTestConditionForm()` | Evaluate a ConditionForm on the actor |
| `DebugIsPlayerLoitering()` | Check loiter detection |
| `DebugSetOnPlayerShip()` | Force on-ship state |
| `DebugExpireFlirtCooldown()` | Clear flirt cooldown |
| `DebugExpireTravelAffinityCoolDown()` | Clear travel-affinity cooldown |

**Usage:** Attach to actor for testing. Optional for release builds.

---

## USING THIS REFERENCE FOR YOUR COMPANION

1. **Understand the Architecture** (Sections 1-6) - How scripts and systems connect
2. **Reference Core Systems** (Sections 7-13) - Affinity, anger, romance, conditions
3. **Actor Scripts** (Section 15) - Scripts attached to the Actor form
4. **API Quick Reference** (Section 14) - Copy/paste ready function calls
5. **Adapt to Your Needs** - Replace placeholder companion names with your character's name

  
All examples show universal patterns. Apply these directly to your companion implementation.  
  
---


