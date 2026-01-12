# Hockey Scorekeeper - AI Assistant Guidelines

## Project Overview
A real-time hockey game scorekeeper built as a **single-file React + Babel application** (`index.html`). Features include:
- **Single-player mode**: localStorage-persisted game state
- **Multiplayer mode**: Firebase Realtime Database sync across editors and viewers
- **Auth system**: Firebase Authentication for game creators/editors
- **Rich stat tracking**: Period breakdowns, player stats (goals, assists, penalties, +/-), goalie records

## Architecture Patterns

### Data Structure
**periodStats**: Period-keyed object tracking team scores, saves, and per-goalie data
```javascript
periodStats[period] = {
  homeScore, awayScore, homeSaves, awaySaves,
  homeGoalie: {starter: {saves, ga}, backup: {saves, ga}},
  awayGoalie: {starter: {saves, ga}, backup: {saves, ga}}
}
```

**playerStats**: Team→players→number→{periods: {}, total: {}}. Stats include G, A, SOG, +/-, FO+/-, penalties.

### Multiplayer Flow
1. **Create**: Generate 6-char code, initialize match in Firebase with `users` list
2. **Join**: Add yourself to `users` (editors) or `viewers` list (read-only)
3. **Sync**: All stat updates push to `matches/{code}` via `matchRef.current.update()`
4. **Disconnect handling**: `onDisconnect().remove()` auto-cleanup; kicked users detected via `kickedUsers` ref
5. **Viewer codes**: Append `-V` to match code; viewers can't edit (checked by `isViewer` flag)

### Single-Player Flow
- Game state stored in localStorage key `juniorLeagueGame` (normalized on load)
- Saves automatically via useEffect dependency on stat changes
- No database sync; state lives in component state + localStorage

### Stat Update Pattern
Key functions handle atomic updates:
- `updateStat(team, stat, delta)`: Period totals (Score/Saves)
- `updatePlayerStat(team, playerNum, stat, delta)`: Player stats, auto-recalculates totals
- When **Score** changes: also updates opposing goalie's GA
- When **Saves** changes: updates active goalie only, recalculates period total

## Critical Implementation Details

### Goalie Tracking
- `activeGoalies` tracks which goalie (starter/backup) is recording for each team
- Goals trigger GA update on **opponent's active goalie**
- Saves only attributed to active goalie; but period save total = starter + backup
- Switch via "Switch Goalie" button in Focus Mode

### Event Handling (Goals/Penalties)
- **Goal events**: Contain team, period, time, situation (EQ/PP/SH), scorers, assists, ±, SOG players
- **Penalty events**: Team, player, type ("2'", "5'", etc.), infraction, time
- Deletion cascades: removing goal event removes associated player stat updates
- Timeline shows reverse chronological with period/team highlighting

### Validation & Permissions
- Editors: require login; max 5 per match (excluding creator); can edit game state
- Viewers: no login needed; read-only (all buttons disabled when `isViewer=true`)
- Creators: can kick editors via `kickedUsers` entry; delete/lock games
- Permission checks: `if(isViewer) return` gates all mutations

### Data Normalization
Calls `normalizePeriodStats()` and `normalizePlayerStats()` on load to handle missing fields—**always safe defaults**. This is critical for DB sync and version compatibility.

## UI/State Management Patterns

### Modals & Wizards
- **Goal wizard**: Multi-step (team selection → scorer/assist selection → save)
- **Penalty wizard**: Simpler form (team, player, type, time, infraction)
- **Add player modal**: Smart comma/space parsing for bulk input (e.g., "2,3,4")
- Collapsible sections: `*Collapsed` state toggles + smooth scroll-into-view on edit

### Focus Mode
- When `focusMode` set (e.g., 'goalies'), renders full-screen goalie editor
- Swappable sides via `goalFocusSwapped` flag
- Exit returns to main tab view

### Dark Mode
- `darkMode` state persisted in localStorage
- Conditional Tailwind classes: `darkMode ? 'bg-gray-900' : 'bg-gradient-to-br from-blue-50 to-blue-100'`

## Firebase Integration

### Database Rules
- `users/{uid}`: only readable/writable by that user
- `matches/{matchId}`: readable by auth users; writable only by creator
  - Editors register at `matches/{code}/users/{uid}` with metadata
  - Viewers at `matches/{code}/viewers/{uid}`
  - Kicked users marked at `matches/{code}/kickedUsers/{uid}`

### Connection Monitoring
- 10-second timeout; if no data received, show error + fallback to single-player
- Real-time listeners via `.on('value', ...)` 
- Connection errors trigger alert + auto-fallback

## Critical Workflows

### Reconnect After Disconnect
- URL param `?join=CODE-V` auto-joins as viewer
- Stored in localStorage as `multiplayerMatch` (with timestamp; auto-expires after 6 hours)
- `setShowReconnectPrompt` offers rejoin option

### Export Functionality
- `generateCSV()`: Period breakdown, goalie stats, per-player period/total rows
- `genReport()`: Human-readable text format
- `downloadCSV()`: Blob download with date-stamped filename

### Sound Management
- Audio refs for homeGoal, awayGoal, homePenalty, awayPenalty, intro (loop)
- `playSound()` resets time, plays, catches errors
- `stopAllSounds()` pauses all + clears state

## Common Edits & Gotchas

1. **Adding new stat type**: Update default objects in `resetGameState()`, normalization functions, CSV export
2. **Changing database structure**: Ensure backward compatibility in normalization—old data must still load
3. **Multiplayer bugs**: Always check `isMultiplayer && matchRef.current` before update; verify auth checks
4. **Goalie logic**: Distinguish between "active goalie" (recording) and "total saves" (both combined)
5. **useEffect deps**: Missing deps cause stale closures; be precise with dependency arrays

## Testing Patterns
- Single-player: Use localStorage in DevTools Console to inspect/reset `juniorLeagueGame`
- Multiplayer: Open two browser tabs with same match code; verify sync
- Permissions: Test viewer restrictions (read-only), kicked user eviction, max editor limits
