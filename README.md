# Starfield Companion Creation Documentation

Community documentation for creating companions in Starfield using the Creation Kit's vanilla companion framework.

## What's Here

This repository contains **documentation only** - no mod files, no copyrighted content.

### Generic Guides (For Any Companion)
| Document | Description |
|----------|-------------|
| `Companion_System_Reference.md` | Technical API reference for Starfield's companion scripts |
| `Companion_Creation_Guide.md` | Step-by-step guide for creating a companion from scratch |

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

## Contributing

Found something wrong? Know something we don't? Open an issue or PR!

## License

Documentation is provided freely for educational purposes. Use it to make cool companions!

---

*Last Updated: 2026-02-01*
