# IcarusBot

Discord bot for Star Citizen org operations: attendance, activity tracking, mission logging, loot raffles, crewed event signups, and aUEC splitting with org tax.

## Setup
- Node 18+
- Create `.env` and set `TOKEN`, `CLIENT_ID`, `GUILD_ID`.
- Install: `npm install`
- Deploy commands: `node deploy-commands.js`
- Run: `node index.js`

## Permissions & Intents
- Scopes: `bot`, `applications.commands`
- Required intents: `Guilds`, `GuildMembers`, `GuildVoiceStates`, `GuildMessages`, `MessageContent`
- Minimum bot perms: View Channels, Send Messages, Embed Links, Read Message History

## Command Reference (slash)

### /eventsquad
Create a squad signup with fixed roles (Squad Lead, Medic) and a variable soldier count.
```
/eventsquad title:"Fireteam Alpha" soldiers:2 details:"Rally on Discord VC"
/eventsquad title:"Fireteam Alpha" soldiers:2 event_id:123   # use event thread anchors; Flex signup is added automatically
```
### /iattendance
- `event date:<YYYY-MM-DD>`: Show attendance for that date's event (guild-scoped). Uses rollups; 10+ min threshold noted.
- `user user:@User [start_date] [end_date]`: Voice time, top channels, messages, and event attendance.
- `summary [start_date] [end_date]`: Event-level totals/averages from rollups.
Examples:
```
/iattendance event date:2025-12-05
/iattendance user user:@Pilot start_date:2025-12-01 end_date:2025-12-07
/iattendance summary start_date:2025-12-01 end_date:2025-12-07
```

### /iaddevent
Add events to the guild database (no immediate post).
```
/iaddevent title:"Free Fly Night" start:"2025-12-16T19:00:00" duration:5 recurrence:weekly topic:"Org Op"
/iaddevent title:"Special Op" start:"2025-12-15T20:00:00" duration:3 recurrence:none topic:"Org Op"
```

### /ilistevents
List all events for this guild.
```
/ilistevents
```

### /iactivity
Guild-scoped rollups from voice/message activity.
- `summary [start_date] [end_date] [limit] [page] [voice_weight] [message_weight] [file]`: Combined leaderboard with weighted score (default voice_weight=1, message_weight=5), paginated. Default limit is 100.
- `user user:@User [start_date] [end_date] [detailed]`: Voice total, top channels, messages. Set `detailed:true` to list each voice session.
- `channel channel:#ch [start_date] [end_date] [limit]`: Top users in a channel.
Examples:
```
/iactivity summary start_date:2025-12-01 end_date:2025-12-07 page:1 limit:10 voice_weight:1 message_weight:0.5
/iactivity user user:@Pilot start_date:2025-12-01 end_date:2025-12-07
/iactivity user user:@Pilot start_date:2025-12-01 end_date:2025-12-07 detailed:true
/iactivity channel channel:#ops start_date:2025-12-01 end_date:2025-12-07
```

### /iautopost
Manually run the daily event autopost for this server (next 14 days).
```
/iautopost
```

### /icommendation
Manage commendations.
- `add user:<name|id> role:<text> reason:<text>`: Add a commendation.
- `view`: Show commendations from the past 30 days.
Examples:
```
/icommendation add user:"Pilot" role:"Medic" reason:"Great triage under fire"
/icommendation view
```

### /ieventconfigure
Configure event autoposting and target channel.
```
/ieventconfigure channel:#ops autopost:true timezone:CST
```

### /ieventcreate
Create a recurring event and post an embed immediately.
```
/ieventcreate title:"Org Op" start:"2025-12-19T19:00:00" duration:2 recurrence:weekly topic:"Org Op" activity:"TBD"
```

### /iinactivity
Uses guild rollups only.
- `list [days] [role]`: Show members with no activity since cutoff.
- `warn [days] [role] [dry_run] [message]`: DM inactive users; `dry_run:true` previews targets.
Examples:
```
/iinactivity list days:14 role:@Member
/iinactivity warn days:30 role:@Member dry_run:true message:"Please check in with the org!"
```

### /imission
Mission night logging (guild-scoped).
- `start name:<text> [notes]`: Begin a mission night.
- `add [mission_id] user:@User amount:<aUEC> [notes]`: Record contribution; defaults to latest open mission.
- `close [mission_id]`: Close mission and total contributions.
- `report [mission_id]`: Show totals with org tax (10%) and per-user payout; records payout recipients.
Examples:
```
/imission start name:"Org Mission Night" notes:"Salvage run"
/imission add user:@Pilot amount:500000
/imission close
/imission report
```

### /ipayoutstatus
List mission nights where recorded participants are still unpaid.
```
/ipayoutstatus
```

### /isplit
Splits a total with 10% org tax, minus sender fees (5% per outgoing transfer). Any remainder goes to org tax.
```
/isplit total:1000000 participants:5
```

### /iraffle
Loot raffle using recent attendance (guild-scoped, deterministic seed).
- `create name:<text> [window_days] [seed]`: Builds weighted entries (voice minutes + messages) over the window (default 7d).
- `entries [raffle_id]`: Show entries for latest or specified raffle.
- `draw [raffle_id]`: Deterministic weighted winner based on stored seed.
Examples:
```
/iraffle create name:"Loot Raffle" window_days:7 seed:"ops-2025-12-01"
/iraffle entries
/iraffle draw
```

### /ieventoneoff
Create an event signup post with dynamic role buttons. Accepts a JSON array or space/comma separated role keys from `role_definitions`.
```
/ieventoneoff title:"Org Op" time:"20:00 CST" roles:'[{"key":"pilot","label":"Pilot","emoji":"🛩️"},{"key":"gunner","label":"Gunner","emoji":"🔫"},{"key":"medic","label":"Medic","emoji":"⛑️"}]' details:"Briefing 15 min early."
/ieventoneoff title:"Org Op" time:"20:00 CST" roles:"pilot gunner engineer" details:"Briefing 15 min early."
```

### /ieventupdate
Update an event instance (title, activity, or details).
```
/ieventupdate event_id:123 title:"Updated Title" activity:"New activity"
```

### /ieventship
Create a signup post based on a ship's configured role capacities (from `ship_roles`). Buttons enforce capacity limits per role.
```
/ieventship ship:"C2 Hercules" time:"20:00 CST" details:"Cargo op"
/ieventship ship:"Idris" event_id:123   # place signup in the event thread; Flex signup added automatically
```

### /ishipid
Find ships by name fragment and show their configured event crew slots (IDs feed into `/ishiproleupdate`).
```
/ishipid name:Idris
```

### /ishiproleupdate
Update the crew slot counts for a ship by ID (only updates provided roles; pilot defaults to 1).
```
/ishiproleupdate ship_id:44 engineer:2 gunner:3
```

### /ishiploadout
Save, view, list, or delete ship loadouts from erkul short URLs (per user, name must be unique).
```
/ishiploadout save name:"Touring Carrack" url:"https://www.erkul.games/loadout/jmbIPMVt" notes:"Touring fit"
/ishiploadout view name:"Touring Carrack"
/ishiploadout list
/ishiploadout delete name:"Touring Carrack"
```

### /imining
Save and test mining loadouts against rock mass/resistance.
- `save name:<text> vehicle_id:<id> config:<laserId:moduleId,moduleId;...>`: Saves a mining loadout (validates vehicle size, laser slots, module existence).
- `calc rock_mass:<int> resistance_pct:<decimal> [loadout_id] [vehicle_id] [config]`: Calculates crackability using saved loadout or ad-hoc config.
Examples:
```
/imining save name:"Mole Power" vehicle_id:1 config:"1:10,11;1:10,11;1:10,11"
/imining calc rock_mass:8000 resistance_pct:0.25 loadout_id:3
/imining calc rock_mass:5000 resistance_pct:0.15 vehicle_id:1 config:"1:10,11;1:10,11;1:10"
```

### /ipermissionsreport
Export server role + channel permission overrides as JSON.
```
/ipermissionsreport mode:summary include_managed_roles:false
```

### /ipoll
Create and manage polls.
- `create question:<text> options:<opt1;opt2;...> [type] [anonymous] [hide_results] [show_percentages] [max_choices] [start_in_minutes] [duration_minutes] [allowed_roles] [allowed_channels] [change_lock_minutes] [description]`
- `close [poll_id] [message_id]`
Examples:
```
/ipoll create question:"Next op?" options:"Salvage;Bounty;Mining" type:single duration_minutes:60 anonymous:true
/ipoll close poll_id:12
```

### /irecord
Record a voice channel to per-user WAV files and upload to Google Drive (server managers only).
```
/irecord join
/irecord stop
/irecord status
/irecord retry_upload
```

### /ihelp and /icommandhelp
- `/ihelp`: Lists commands with short descriptions. Use `/icommandhelp` for details.
- `/icommandhelp cmd:<name>`: Detailed options/subcommands/examples for one command.

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

## Recording Setup (Google Drive)
Set these in `.env` for `/irecord` uploads:
- `GOOGLE_CLIENT_ID`
- `GOOGLE_CLIENT_SECRET`
- `GOOGLE_REFRESH_TOKEN`
- `GOOGLE_DRIVE_FOLDER_ID` (optional parent folder for uploads)
- `GOOGLE_DRIVE_SHARE_PUBLIC` (set to `true` to share links publicly; leave unset for private links)
- `RECORDINGS_TMP_DIR` (optional temp folder; default `recordings/tmp`)
- `RECORDINGS_MAX_HOURS` (auto-stop duration, default 6)

Refresh token notes (OAuth personal account):
- Create an OAuth client in Google Cloud Console (Desktop app or Web app).
- Use the OAuth Playground to exchange an auth code for a refresh token.
- Add the refresh token to `.env` as `GOOGLE_REFRESH_TOKEN`.
- If upload fails or Drive is not configured, WAVs remain in `recordings/tmp/<sessionId>`.
- Recording auto-stops after `RECORDINGS_MAX_HOURS` or when the channel is empty of non-bot users.
- Use `/irecord retry_upload` to retry the most recent upload or provide a `session_id` (the temp folder name).
