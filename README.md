# IcarusBot

Discord bot for Star Citizen org operations: attendance, activity tracking, mission logging, loot raffles, crewed event signups, and aUEC splitting with org tax.

## Setup
- Node 18+
- Copy `.env.example` to `.env` and set `TOKEN`, `CLIENT_ID`, `GUILD_ID`.
- Install: `npm install`
- Deploy commands: `node deploy-commands.js`
- Run: `node index.js`

## Permissions & Intents
- Scopes: `bot`, `applications.commands`
- Required intents: `Guilds`, `GuildMembers`, `GuildVoiceStates`, `GuildMessages`, `MessageContent`
- Minimum bot perms: View Channels, Send Messages, Embed Links, Read Message History

## Command Reference (slash)
### /attendance
- `event date:<YYYY-MM-DD>`: Show attendance for that date’s event (guild-scoped). Uses rollups; 10+ min threshold noted.
- `user user:@User [start_date] [end_date]`: Voice time, top channels, messages, and event attendance.
- `summary [start_date] [end_date]`: Event-level totals/averages from rollups.
Examples:
```
/attendance event date:2025-12-05
/attendance user user:@Pilot start_date:2025-12-01 end_date:2025-12-07
/attendance summary start_date:2025-12-01 end_date:2025-12-07
```

### /addevent
Add events to the guild.
```
/addevent name:"Free Fly Night" start_time:19:00 end_time:00:00 recurrence:weekly day_of_week:Tuesday
/addevent name:"Special Op" start_time:20:00 end_time:23:00 recurrence:none date:2025-12-15
```

### /listevents
List all events for this guild.
```
/listevents
```

### /activity
Guild-scoped rollups from voice/message activity.
- `summary [start_date] [end_date] [limit] [page] [voice_weight] [message_weight]`: Combined leaderboard with weighted score (default 1:1), paginated.
- `user user:@User [start_date] [end_date]`: Voice total, top channels, messages.
- `channel channel:#ch [start_date] [end_date] [limit]`: Top users in a channel.
Examples:
```
/activity summary start_date:2025-12-01 end_date:2025-12-07 page:1 limit:10 voice_weight:1 message_weight:0.5
/activity user user:@Pilot start_date:2025-12-01 end_date:2025-12-07
/activity channel channel:#ops start_date:2025-12-01 end_date:2025-12-07
```

### /inactivity
Uses guild rollups only.
- `list [days] [role]`: Show members with no activity since cutoff.
- `warn [days] [role] [dry_run] [message]`: DM inactive users; `dry_run:true` previews targets.
Examples:
```
/inactivity list days:14 role:@Member
/inactivity warn days:30 role:@Member dry_run:true message:"Please check in with the org!"
```

### /mission
Mission night logging (guild-scoped).
- `start name:<text> [notes]`: Begin a mission night.
- `add [mission_id] user:@User amount:<aUEC> [notes]`: Record contribution; defaults to latest open mission.
- `close [mission_id]`: Close mission and total contributions.
- `report [mission_id]`: Show totals with org tax (10%) and per-user payout; records payout recipients.
Examples:
```
/mission start name:"Org Mission Night" notes:"Salvage run"
/mission add user:@Pilot amount:500000
/mission close
/mission report
```

### /payoutstatus
List mission nights where recorded participants are still unpaid.
```
/payoutstatus
```

### /split
Splits a total with 10% org tax, even split remainder.
```
/split total:1000000 participants:5
```

### /raffle
Loot raffle using recent attendance (guild-scoped, deterministic seed).
- `create name:<text> [window_days] [seed]`: Builds weighted entries (voice minutes + messages) over the window (default 7d).
- `entries [raffle_id]`: Show entries for latest or specified raffle.
- `draw [raffle_id]`: Deterministic weighted winner based on stored seed.
Examples:
```
/raffle create name:"Loot Raffle" window_days:7 seed:"ops-2025-12-01"
/raffle entries
/raffle draw
```

### /eventpost
Create an event signup post with dynamic role buttons. Accepts a JSON array or space/comma separated role keys from `role_definitions`.
```
/eventpost title:"Org Op" time:"20:00 CST" roles:'[{"key":"pilot","label":"Pilot","emoji":"🛩️"},{"key":"gunner","label":"Gunner","emoji":"🔫"},{"key":"medic","label":"Medic","emoji":"⛑️"}]' details:"Briefing 15 min early."
/eventpost title:"Org Op" time:"20:00 CST" roles:"pilot gunner engineer" details:"Briefing 15 min early."
```

### /eventship
Create a signup post based on a ship's configured role capacities (from `ship_roles`). Buttons enforce capacity limits per role.
```
/eventship ship:"C2 Hercules" time:"20:00 CST" details:"Cargo op"
```

### /shiploadout
Save, view, list, or delete ship loadouts from erkul short URLs (per user, name must be unique).
```
/shiploadout save name:"Touring Carrack" url:"https://www.erkul.games/loadout/jmbIPMVt" notes:"Touring fit"
/shiploadout view name:"Touring Carrack"
/shiploadout list
/shiploadout delete name:"Touring Carrack"
```

### /mining
Save and test mining loadouts against rock mass/resistance.
- `save name:<text> vehicle_id:<id> config:<laserId:moduleId,moduleId;...>`: Saves a mining loadout (validates vehicle size, laser slots, module existence).
- `calc rock_mass:<int> resistance_pct:<decimal> [loadout_id] [vehicle_id] [config]`: Calculates crackability using saved loadout or ad-hoc config.
Examples:
```
/mining save name:"Mole Power" vehicle_id:1 config:"1:10,11;1:10,11;1:10,11"
/mining calc rock_mass:8000 resistance_pct:0.25 loadout_id:3
/mining calc rock_mass:5000 resistance_pct:0.15 vehicle_id:1 config:"1:10,11;1:10,11;1:10"
```

### /help and /commandhelp
- `/help [page]`: Lists commands with short descriptions (paged). Use `/commandhelp` for details.
- `/commandhelp command:<name>`: Detailed options/subcommands/examples for one command.

## Event & Activity Behavior
- Voice joins/leaves are logged and rolled up per day/channel.
- Message counts are tracked per day/channel.
- Startup maintenance auto-closes stale voice sessions if members are no longer in that channel and backfills rollups.
- All stats/commands are guild-scoped; legacy rows without `guild_id` are still readable but new data is scoped.

## Data Tables (SQLite)
- `voice_sessions`: raw joins/leaves
- `voice_activity_daily`: per-user/channel/day seconds
- `message_counts`: per-user/channel/day count
- `events`: scheduled events (guild_id)
- `event_attendance`: per-event occurrence attendance
- `mission_nights`, `mission_participants`: mission logging (guild_id)
- `payouts`, `payout_recipients`: splits/logs (guild_id)
- `raffles`, `raffle_entries`: weighted raffle tracking (guild_id)
- `ship_roles`: per-ship role capacities
- `role_definitions`: role metadata used by signup posts
- `ship_loadouts` and related component tables for imported loadouts

## Deployment
- Update slash commands after changes: `node deploy-commands.js`
- Ensure bot has required intents/permissions in the Discord developer portal.
- Optional guild backfill for legacy rows (events/mission/payouts/raffles): `npm run migrate:guild` (requires `GUILD_ID` in `.env`).
