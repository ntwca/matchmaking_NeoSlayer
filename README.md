const Express = require("express");
const express = Express.Router();
const fs = require("fs");
const path = require("path");
const iniparser = require("ini");
const config = iniparser.parse(fs.readFileSync(path.join(__dirname, "..", "Config", "config.ini")).toString());
const functions = require("./functions.js");
const lawinSignals = require("./lawinClientSignals.js");

// Used by NeoSlayer client (HTTP poll) during "searching for match" (Jouer).
let lawinMatchmakingPartySearchActive = false;
let lawinMatchmakingLastSearchPulseMs = 0;
const LAWIN_SEARCH_ACTIVE_WINDOW_MS = 300000;

function refreshLawinMatchmakingSearchPulse() {
    lawinMatchmakingPartySearchActive = true;
    lawinMatchmakingLastSearchPulseMs = Date.now();
    try {
        lawinSignals.onTryPlayOnPlatform();
    } catch (e) {
        /* optional */
    }
}

function lawinMatchmakingSearchingForClient() {
    if (!lawinMatchmakingPartySearchActive)
        return false;
    return Date.now() - lawinMatchmakingLastSearchPulseMs < LAWIN_SEARCH_ACTIVE_WINDOW_MS;
}

express.get("/lawin/internal/matchmaking/searching", (req, res) => {
    res.type("application/json");
    res.json({
        searching: lawinMatchmakingSearchingForClient(),
        tryPlayClick: lawinSignals.tryPlayClickActiveForClient(),
        tryPlaySeq: lawinSignals.getTryPlaySeq(),
    });
});

function matchmakingServiceUrl() {
    const gsIp = (config.GameServer && config.GameServer.ip) ? String(config.GameServer.ip).trim() : "";
    const ms = config.MatchmakingService || {};
    const msIp = ms.ip != null && String(ms.ip).trim() !== "" ? String(ms.ip).trim() : "";
    const host = msIp || gsIp || "127.0.0.1";
    let port = Number(ms.port != null && String(ms.port).trim() !== "" ? ms.port : 80);
    if (!Number.isFinite(port) || port <= 0) port = 80;
    if (port !== 80) return `ws://${host}:${port}`;
    return `ws://${host}`;
}

express.get("/fortnite/api/matchmaking/session/findPlayer/*", async (req, res) => {
    res.status(200);
    res.end();
})

express.get("/fortnite/api/game/v2/matchmakingservice/ticket/player/*", async (req, res) => {
    refreshLawinMatchmakingSearchPulse();
    res.cookie("currentbuildUniqueId", req.query.bucketId.split(":")[0]);

    res.json({
        "serviceUrl": matchmakingServiceUrl(),
        "ticketType": "mms-player",
        "payload": "69=",
        "signature": "420="
    })
    res.end();
})

express.get("/fortnite/api/game/v2/matchmaking/account/:accountId/session/:sessionId", async (req, res) => {
    res.json({
        "accountId": req.params.accountId,
        "sessionId": req.params.sessionId,
        "key": "AOJEv8uTFmUh7XM2328kq9rlAzeQ5xzWzPIiyKn2s7s="
    })
})

express.get("/fortnite/api/matchmaking/session/:session_id", async (req, res) => {
    res.json({
        "id": req.params.session_id,
        "ownerId": functions.MakeID().replace(/-/ig, "").toUpperCase(),
        "ownerName": "[DS]fortnite-liveeugcec1c2e30ubrcore0a-z8hj-1968",
        "serverName": "[DS]fortnite-liveeugcec1c2e30ubrcore0a-z8hj-1968",
        "serverAddress": config.GameServer.ip,
        "serverPort": Number(config.GameServer.port),
        "maxPublicPlayers": 220,
        "openPublicPlayers": 175,
        "maxPrivatePlayers": 0,
        "openPrivatePlayers": 0,
        "attributes": {
          "REGION_s": "EU",
          "GAMEMODE_s": "FORTATHENA",
          "ALLOWBROADCASTING_b": true,
          "SUBREGION_s": "GB",
          "DCID_s": "FORTNITE-LIVEEUGCEC1C2E30UBRCORE0A-14840880",
          "tenant_s": "Fortnite",
          "MATCHMAKINGPOOL_s": "Any",
          "STORMSHIELDDEFENSETYPE_i": 0,
          "HOTFIXVERSION_i": 0,
          "PLAYLISTNAME_s": "Playlist_DefaultSolo",
          "SESSIONKEY_s": functions.MakeID().replace(/-/ig, "").toUpperCase(),
          "TENANT_s": "Fortnite",
          "BEACONPORT_i": 15009
        },
        "publicPlayers": [],
        "privatePlayers": [],
        "totalPlayers": 45,
        "allowJoinInProgress": false,
        "shouldAdvertise": false,
        "isDedicated": false,
        "usesStats": false,
        "allowInvites": false,
        "usesPresence": false,
        "allowJoinViaPresence": true,
        "allowJoinViaPresenceFriendsOnly": false,
        "buildUniqueId": req.cookies.currentbuildUniqueId || "0", // buildUniqueId is different for every build, this uses the netver of the version you're currently using
        "lastUpdated": new Date().toISOString(),
        "started": false
      })
})

express.post("/fortnite/api/matchmaking/session/*/join", async (req, res) => {
    lawinMatchmakingPartySearchActive = false;
    res.status(204);
    res.end();
})

express.post("/fortnite/api/matchmaking/session/matchMakingRequest", async (req, res) => {
    refreshLawinMatchmakingSearchPulse();
    res.json([])
})

module.exports = express;
