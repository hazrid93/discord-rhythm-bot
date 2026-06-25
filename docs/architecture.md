# 0xRhythm-Bot — Architecture

> Node.js Discord audio bot streaming from YouTube and Soundcloud via `play-dl`. Features per-guild priority queue, user/guild play history in MongoDB, and TTS support. Deployed via PM2 with Docker support.

| | |
|---|---|
| **Language** | TypeScript · CommonJS |
| **Runtime** | Node.js 16.17.x |
| **Discord** | discord.js v14 · @discordjs/voice · @discordjs/opus |
| **Audio** | play-dl (YouTube + Soundcloud) · fluent-ffmpeg |
| **Database** | MongoDB via Mongoose |
| **Process** | PM2 (dev + prod) · Docker (Containerfile + docker-compose) |
| **TTS** | node-gtts |

---

## High-Level Architecture

```mermaid
flowchart TB
  subgraph Discord["Discord Gateway"]
    GUILD["Guild Voice Channel"]
    USER["User sends slash command"]
  end

  subgraph Bot["Bot Process  (PM2)"]
    CLIENT["discord.js Client<br/>GatewayIntentBits"]
    APP["app.ts<br/>interactionCreate handler"]
    SUBS["subscriptions Map<br/>guildId to Playlist"]
    HANDLERS["Handlers<br/>YoutubeHandler<br/>SoundcloudHandler<br/>TextToSpeechHandler"]
  end

  subgraph PlaylistLayer["Per-guild Playlist"]
    PL["Playlist<br/>priority queue + audio config"]
    WORKER["Worker Thread<br/>child_player.ts"]
    QUEUE["PriorityQueue<br/>binary max-heap"]
  end

  subgraph Player["Child Player  (Worker Thread)"]
    DL["play-dl<br/>stream audio from source"]
    FFMPEG["fluent-ffmpeg<br/>bass and treble filters"]
    AUDIO["AudioPlayer<br/>createAudioResource"]
    VOICE["VoiceConnection<br/>joinVoiceChannel"]
  end

  subgraph DB["MongoDB"]
    GUILDDB["guild.model<br/>guild history"]
    USERDB["user.model<br/>user play history"]
  end

  USER --> CLIENT
  CLIENT --> APP
  APP --> SUBS
  APP --> HANDLERS
  HANDLERS --> PL
  PL --> QUEUE
  PL --> WORKER
  WORKER --> DL
  DL --> FFMPEG
  FFMPEG --> AUDIO
  AUDIO --> VOICE
  VOICE --> GUILD
  APP --> GUILDDB
  APP --> USERDB
```

---

## Slash Command Flow

```mermaid
sequenceDiagram
  participant User as Discord User
  participant Client as discord.js Client
  participant App as app.ts
  participant Subs as subscriptions Map
  participant PL as Playlist
  participant Worker as child_player.ts
  participant Source as YouTube or Soundcloud

  User->>Client: slash command (play, skip, clear, etc.)
  Client->>App: interactionCreate event
  App->>App: check isChatInputCommand + guildId

  alt play command
    App->>Subs: get(guildId)
    alt no subscription exists
      App->>Subs: create new Playlist(guildId)
      App->>PL: create guild in MongoDB
    end
    App->>Source: search or resolve URL via play-dl
    Source-->>App: track info
    App->>PL: enqueue Track with priority
    PL->>PL: PriorityQueue.enqueue (bubbleUp)
    alt player idle
      PL->>Worker: spawn Worker Thread with track data
      Worker->>Source: play-dl.stream(url)
      Source-->>Worker: audio stream
      Worker->>Worker: ffmpeg bass and treble filters
      Worker->>Worker: createAudioResource + play
    end
    App-->>User: embed response with queue position
  else skip, clear, pause, resume, leave
    App->>PL: call command method
    PL->>Worker: IPC message (IPC_STATES_REQ)
    Worker-->>PL: IPC response (IPC_STATES_RESP)
    App-->>User: confirmation embed
  end
```

---

## Priority Queue — Binary Max-Heap

Tracks are ordered by `TrackPriority` (0=LOW, 1, 2 — higher overrides queue position). Ties are broken by `creationDate` (newer first). The max element is guaranteed at the root.

```mermaid
flowchart TD
  ENQ["enqueue Track"] --> PUSH["push to end of array"]
  PUSH --> BUBBLE["bubbleUp"]
  BUBBLE --> COMPARE{"element.priority ><br/>parent.priority?"}
  COMPARE -- "element <= parent" --> DONE1["correct position — done"]
  COMPARE -- "element > parent" --> SWAP["swap with parent"]
  SWAP --> BUBBLE

  DEQ["dequeue"] --> ROOT["take root (max priority)"]
  ROOT --> REPLACE["move last element to root"]
  REPLACE --> SINK["sinkDown"]
  SINK --> COMPARE2{"element.priority ><br/>larger child?"}
  COMPARE2 -- "element >= larger child" --> DONE2["correct position — done"]
  COMPARE2 -- "element < larger child" --> SWAP2["swap with larger child"]
  SWAP2 --> SINK
```

---

## Track Playback Lifecycle

```mermaid
sequenceDiagram
  participant PL as Playlist
  participant Worker as child_player.ts
  participant DL as play-dl
  participant VP as VoiceConnection
  participant AP as AudioPlayer

  PL->>Worker: spawn Worker Thread with track data
  Worker->>Worker: decode base64 workerData
  Worker->>VP: joinVoiceChannel(guildId, channelId)
  VP-->>Worker: VoiceConnection ready

  Worker->>DL: play-dl.stream(url)
  DL-->>Worker: audio stream + type
  Worker->>AP: createAudioResource(stream)
  Worker->>AP: AudioPlayer.play(resource)
  AP-->>VP: audio piped to Discord

  loop until track ends
    AP->>AP: AudioPlayerStatus.Playing
  end

  AP-->>Worker: AudioPlayerStatus.Idle
  Worker->>PL: IPC message: track finished
  PL->>PL: dequeue next track from PriorityQueue
  alt queue not empty
    PL->>Worker: spawn next track
  else queue empty
    PL->>PL: cleanup, remove subscription
  end
```

---

## IPC Between Main and Worker Thread

The main process (`app.ts`) and the child player (`child_player.ts`) communicate via Node.js `worker_threads` IPC messages, defined in `constants/ipcStates.ts`.

```mermaid
flowchart LR
  subgraph Main["Main Process  (app.ts)"]
    PL["Playlist"]
  end

  subgraph Child["Worker Thread  (child_player.ts)"]
    PLAYER["AudioPlayer"]
    VOICE["VoiceConnection"]
  end

  PL -- "IPC_STATES_REQ" --> PLAYER
  PLAYER -- "IPC_STATES_RESP" --> PL

  subgraph Requests["IPC_STATES_REQ"]
    R1["PLAY_TRACK"]
    R2["PAUSE"]
    R3["RESUME"]
    R4["SKIP"]
    R5["STOP"]
    R6["SET_AUDIO_CONFIG<br/>bass, treble, volume"]
  end

  subgraph Responses["IPC_STATES_RESP"]
    S1["TRACK_FINISHED"]
    S2["PLAYER_ERROR"]
    S3["VOICE_DISCONNECTED"]
    S4["STATUS_UPDATE"]
  end

  Requests --> CHILD_IPC["worker.postMessage"]
  Responses --> MAIN_IPC["parentPort.on message"]
```

---

## Deployment Topology

```mermaid
flowchart TB
  subgraph Dev["Development"]
    DEV_ENV[".env.local<br/>DB_URL, STAGING_TOKEN, CLIENT_ID"]
    PM2_DEV["PM2: 0xRythm-DEV<br/>npm run start"]
  end

  subgraph Prod["Production"]
    PROD_ENV[".env.prod<br/>DB_URL, STAGING_TOKEN, CLIENT_ID"]
    PM2_PROD["PM2: 0xRythm-PROD<br/>npm run start:prod"]
    LOGROTATE["pm2-logrotate<br/>100MB max log size"]
  end

  subgraph Docker["Docker"]
    CONTAINERFILE["Containerfile"]
    COMPOSE["docker-compose.yaml<br/>docker-compose.dev.yaml"]
  end

  subgraph Mongo["MongoDB"]
    DB["mongodb://localhost:27017<br/>maxPoolSize=100, w=majority"]
    DB --> GUILD_COLL["guilds collection"]
    DB --> USER_COLL["users collection"]
  end

  DEV_ENV --> PM2_DEV --> DB
  PROD_ENV --> PM2_PROD --> DB
  CONTAINERFILE --> COMPOSE
  PM2_DEV --> LOGROTATE
  PM2_PROD --> LOGROTATE
```
