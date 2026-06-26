# 0xRhythm-Bot

Node.js | Discord audio bot featuring play-dl to stream audio from YouTube & Soundcloud. Supports user/guild history using MongoDB and priority queue feature for playlist.

## Architecture

- 📎 **[Architecture & Sequence Diagrams](docs/architecture.md)** — flow, command lifecycle, priority queue, IPC, and deployment topology
- [DeepWiki architecture document](https://deepwiki.com/hazrid93/0xRhythm-Bot)
## Invite link for bot im running on my instance
- inv url [link](https://discord.com/api/oauth2/authorize?client_id=935569503729897562&permissions=1084516956992&scope=bot%20applications.commands)

## Requirement
-Node: v16.17.x <br/>

## Description
Simple little music bot to queue up and play youtube audio over discord voice channels.

## Bot Commands
```
/play 
    - <soundcloud|youtube: Select the track provider.> 
    - <value|song name: Url of track or song name to search for.> 
    - <priority: 0, 1, 2 .Higher means it will override the queue position.>
/clear Clear the current playlist in the guild.
/skip Skip the current track in the guild playlist.
/leave Boot the player from the voice channel, this will clear the playlist.
/pause Pause the current track that is playing.
/resume Resume the current track that is playing.
/queue Prints the current playlist in the guild.
/status Get status of the server.
/tts <text: text that you want TTS to transmit to current user channel> 
/config 
    - <bass: 0-20 .Set the bass value for the subsequent track in playlist.>
    - <treble: 0-20 .Set the treble value for the subsequent track in playlist.>
```
### Running the Application
    Note: 
    1) Setup .env.local for development server and .env.prod for production server on project
    root directory.
    Example .env.local :
        VERSION=1.0.0
        DB_URL=mongodb://localhost:27017/?maxPoolSize=100&w=majority
        DB_NAME=local-test
        STAGING_TOKEN=<token from discord dev portal>
        CLIENT_ID=<bot client id>
    
    2) Run the commands below (or refer package.json)
    #### [DEVELOPMENT SERVER]
-   Run `npm run start:pm2:dev` - start the server under pm2 process manager
-   Run `npm run logs:pm2` - tail the log for the process in realtime
-   Run `npm run delete:pm2:dev` - Kill the server process
-   Run `npm run monitor:pm2` - Kill the server process

    #### [PRODUCTION SERVER]
-   Run `npm run start:pm2:prod` - start the server under pm2 process manager
-   Run `npm run logs:pm2` - tail the log for the process in realtime
-   Run `npm run delete:pm2:prod` - Kill the server process
-   Run `npm run monitor:pm2` - Kill the server process


---

## Key Features & Highlights

0xRhythm-Bot is a Discord audio bot that streams from YouTube and Soundcloud via `play-dl`, persists per-user/per-guild listening history in MongoDB, and runs each playback as an isolated Node.js worker thread. The engineering choices below are what distinguish it from a typical "play command" bot.

- **Binary max-heap priority queue** (`src/utils/priorityqueue.ts`) — Instead of a simple FIFO array, the playlist is backed by a hand-rolled binary max-heap. `enqueue` pushes to the array tail and runs `bubbleUp` (lines 18–36), which walks toward the root swapping when `element.priority > parent.priority` (ties broken by `creationDate`). `dequeue` pops the root, moves the last element there, and restores the heap invariant via `sinkDown` (lines 76–124), comparing against both children with the `2*idx+1` / `2*idx+2` index formula. This gives O(log n) priority insertion and extraction rather than O(n) array re-sorts, and `printQueue` (lines 46–59) builds a readable ordered list by cloning the array and repeatedly dequeuing without mutating the live heap.

- **Three-tier `TrackPriority` overriding queue position** (`src/utils/trackpriority.ts`, `src/handlers/youtube.handler.impl.ts`) — Tracks declare `LOW=0`, `MEDIUM=1`, or `HIGH=2`. The `/play` slash command makes priority a required option, and the heap weights these so a `HIGH` track jumps ahead of lower-priority enqueued items. The queue's tie-break on `creationDate` (newer wins) means equal-priority tracks still preserve a deterministic FIFO-like order — a deliberate, non-obvious design choice in `bubbleUp`/`sinkDown`.

- **Per-guild audio playback in isolated worker threads** (`src/player/child_player.ts`, `src/playlist/playlist.main.ts:267-328`) — Each `Playlist` spawns a `new Worker(__dirname + './../player/child_player.ts', { workerData })` (line 317) rather than streaming in the main process. The track payload (`ForkObject`) is JSON-serialized and **base64-encoded** before being passed across the IPC boundary (lines 288, 314), and the child decodes it on `ready`. This isolates `play-dl` streaming, ffmpeg filter chains, and `@discordjs/voice` `VoiceConnection`/`AudioPlayer` state from the main command router — a crash in one guild's stream cannot take down the bot.

- **Typed IPC protocol between main and worker** (`src/constants/ipcStates.ts`) — Communication uses two string-enum contracts: `IPC_STATES_REQ` (`PAUSE_VOICE_CONNECTION`, `RESUME_VOICE_CONNECTION`, `SKIP_VOICE_CONNECTION`, `LEAVE_VOICE_CONNECTION`, `START_PLAYER_PROCESS`, `GET_VOICE_CONNECTION_READINESS`) and `IPC_STATES_RESP` (`SONG_IDLE`, `SONG_PLAYING`, `SONG_PAUSED`, `QUEUE_EMPTY`, `REMOVE_GUILD_SUBSCRIPTION`, `VOICE_CONNECTION_READY`). The parent dispatches responses in `Playlist.commandHandler` (lines 330–360) to drive `processQueue` on `SONG_IDLE` and tear down the subscription on `REMOVE_GUILD_SUBSCRIPTION`, while the child handles requests in `parentPort.on('message')` (lines 259–305) including a `default` branch that re-decodes a base64 track and re-enters `execute`.

- **Robust voice-disconnect recovery with bounded rejoin** (`src/player/child_player.ts:200-253`) — The `VoiceConnection` `stateChange` handler implements Discord's recommended recovery state machine. On a WebSocket close `4014` (kicked vs. moved), it waits up to 5 seconds via `entersState(..., Connecting, 5_000)` to disambiguate; if it can't reconnect, it posts `REMOVE_GUILD_SUBSCRIPTION` and exits. For other disconnects it caps rejoin attempts at `rejoinAttempts < 5` with backoff `(rejoinAttempts + 1) * 5_000` ms, and on `Signalling`/`Connecting` states it applies a 20-second readiness deadline guarded by a `readyLock` flag so the connection can't live-stuck in a transient state. This is the canonical `@discordjs/voice` resilience pattern, correctly implemented.

- **Strategy-based provider handlers** (`src/handlers/player.strategy.ts`, `youtube.handler.impl.ts`, `soundcloud.handler.impl.ts`, `tts.handler.impl.ts`) — Each source implements a `PlayStrategy { execute(): Promise<boolean> }` interface. The YouTube handler regex-tests for `^(http|https)`, and if absent calls `playdl.search(name, { limit: 1, source: { youtube: "video" } })` to auto-resolve a song title to a URL. It distinguishes `yt_validate` returning `'video'` vs `'playlist'` — for playlists it uses `playlist_info(url, { incomplete: true })` to skip hidden videos and iterates `all_videos()`. Soundcloud mirrors this with `getFreeClientID()` token setup and `soundcloud(url)` type-checking. The TTS handler enqueues a `Track` with `null` url/provider and `TrackPriority.HIGH` so announcements preempt music.

- **Per-track ffmpeg bass/treble DSP via `audioConfig`** (`src/playlist/playlist.main.ts:133-161`, `audioConfig.ts`) — `Playlist` holds an `AudioConfig { volume, bass, treble }` clamped 0–20 by `setBass`/`setTreble` (lines 108–128). `getAudioConfigFfmpeg()` serializes this into ffmpeg `-af` arguments — `volume=`, `bass=g=`, `treble=g=` — passed as a `string[]` on every track fork. The code also retains commented-out examples of advanced filters (`firequalizer` multi-band gain entries, `chorus` effect) showing the DSP pipeline was designed for richer audio shaping. The config applies to "the subsequent track," decoupling EQ changes from the currently-playing stream.

- **MongoDB-backed user/guild listening history with bounded track arrays** (`src/models/user.model.ts`, `src/playlist/playlist.main.ts:177-227`, `src/constants/userConfig.ts`) — On enqueue, `updateUser` finds-or-creates a `User` document (unique `userId`) and pushes the track to its `tracks[]` subdocument array. A `TRACK_LIMIT` of 1000 (`userConfig.ts`) triggers `newTrack.shift()` when the array exceeds the cap (line 213), implementing a circular-history ring buffer per user so history never grows unbounded. Guilds reference users via `ObjectId` (`ref: 'User'`) and vice-versa (`ref: 'Guild'`), and on first user-play in a guild the bot back-links the persisted user into the guild's `users[]` array (lines 194–202).


- **MongoDB data layer with soft-delete hooks and tuned connection pooling** (`src/models/guild.model.ts`, `src/utils/mongodbutils.ts`) — Both `User` and `Guild` schemas register `pre('find')` and `pre('findOne')` hooks injecting `this.where({ isDeleted: false })`, so every read automatically filters tombstoned documents without callers needing the predicate; combined with `timestamps: true` and custom collection names (`Guilds`, `Users`, `Tracks`). Connections are established with `maxPoolSize: 100` and `autoIndex: true` (mirroring the `?maxPoolSize=100&w=majority` URI), and a cached `_connection` singleton in `getConnection()` avoids reconnect-per-query overhead. `connectToServer` uses a callback gate so the main process refuses to boot (exits 1) unless the DB is reachable — fail-fast at startup rather than silent degraded operation.

- **TTS as a first-class high-priority queue citizen** (`src/handlers/tts.handler.impl.ts`, `src/player/child_player.ts:160-170`) — `/tts` enqueues a track with `url=null` and `TrackPriority.HIGH`, routed by the child: when `url == null`, `execute` builds an `AudioResource` from `gtts.stream(textTTS)` with `inputType: StreamType.Arbitrary` instead of calling `play-dl`. Combined with the heap priority, TTS announcements naturally preempt the music queue and the child's `final: true` flag lets a queue-empty TTS message fire `QUEUE_EMPTY` instead of advancing the queue.

- **Idle-voice auto-leave and worker lifecycle hygiene** (`src/player/child_player.ts:80-94, 183-198`) — On `AudioPlayerStatus.Idle`, the child sets a `setTimeout` of `10*60*1000` ms (10 minutes) to `exit()`, which removes all `AudioPlayer`/`VoiceConnection` listeners, destroys the connection, and calls `process.exit(0)` — preventing zombie voice connections when a queue drains. The `exit()` function defensively wraps listener removal in try/catch so a partially-initialized player still cleans up. `Playlist.startProcessEventListener` (lines 52–81) mirrors this on the parent side, hooking `SIGTERM`/`SIGINT`/`exit` to `childProcess.terminate()` and `process.exit(0)` for clean PM2 shutdowns.
