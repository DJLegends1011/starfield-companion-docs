# Starfield Companion Creation Documentation

Community documentation for creating companions in Starfield using the Creation Kit's vanilla companion framework.

## What's Here

This repository contains **documentation only** - no mod files, no copyrighted content.

### Generic Guides (For Any Companion)
| Document | Description |
|----------|-------------|
| `Companion_System_Reference.md` | Technical API reference for Starfield's companion scripts |
| `Companion_Creation_Guide.md` | Step-by-step guide for creating a companion from scratch |
| `DRAFT_Affinity_Events_Section.md` | Affinity Events design section (worksheet format) written for *Aflin & DJLegend's Starfield NPC Template* — fire conditions verified against the vanilla Papyrus sources |
| `CompanionAffinityEvents.csv` | Data dump of every vanilla affinity event form (~46 Action, ~319 CommentTrigger, plus DialogueResponse/WantsToTalk/Direct) with descriptions and per-companion reactions — source data for the section above |

### Aflin & DJLegend's Starfield NPC Template
The Affinity Events section here is our contribution to the **Starfield NPC Template**, a designer-to-scripter worksheet co-authored with [Aflin](https://www.nexusmods.com/skyrimspecialedition/mods/138633) (author of the original Skyrim NPC Template). **Aflin will be releasing the Starfield template on Nexus soon** — once it's live, a link will be added here. The three docs are designed to work together: the Template covers *what to write*, the Creation Guide *how to build it*, and the System Reference *why it works*.

### Example Implementation
Example implementation available when Nexus release of my companion comes out.

## Why This Exists

Bethesda built a robust companion system for Starfield that vanilla companions barely utilize. These docs aim to:

1. **Reverse-engineer** how the system actually works
2. **Document** the scripts, properties, and systems involved
3. **Provide** a practical example (Cass) that others can learn from
4. **Enable** modders to create companions equal to or better than vanilla

## Key Technical Discoveries

- **Anger System:** `AmIAngry()` checks for AngerLevel >= 2, so Level 1 (Annoyed) won't trigger auto-dismiss
- **WantsToTalk:** `COM_CND_WantsToTalk_Standard` is shared by all companions
- **Story Gates:** Vanilla provides `COM_StoryGate_TimerDuration_01-08_Standard` GlobalVariables
- **Distance:** Starfield uses meters (not old Bethesda units)
- **Crime Response:** Personal Crime Factions control which NPCs companions consider "civilians"
- **Dead event:** `COM_Event_Action_UseWorkbench` never fires in vanilla — its handler is commented out in `CompanionAffinityEventsScript.psc` (bug GEN-432521)
- **Unreachable event:** `COM_Event_Action_ZeroG` can't fire via the automatic gravity check (the below-0.5g branch catches zero gravity first as GravityLow); quests must send it directly
- **Affinity throttling:** all action events share one global cooldown, and events are dropped (not queued) during dialogue and scenes; Steal/Pickpocket also require the companion to have line-of-sight on the player

## Scripts Documented

- `SQ_CompanionsScript` - Full companion management
- `SQ_FollowersScript` - Basic following behavior
- `CompanionActorScript` - Actor-attached companion functions
- `CompanionAffinityScript` - Affinity and anger events
- `CompanionCrimeResponseScript` - Crime response behavior
- `COM_CompanionQuestScript` - Personal quest management

## Credits

- **Craftian** - Original Cass character concept
- **Demetri (DJLegnds)** - Creation Kit implementation & testing
- **Claude** - Documentation & dialogue writing
- **Aflin** - Original NPC Template format & Starfield Template co-author

## Contributing

Found something wrong? Know something we don't? Open an issue or PR!

## License

Documentation is provided freely for educational purposes. Use it to make cool companions!

---

*Last Updated: 2026-07-01*
