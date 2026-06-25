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
