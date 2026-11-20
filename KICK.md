# Podpora Kicku v Ganymede (fork corekill/ganymede, větev `kick-support`)

Fork vychází z upstream **v5.0.3** (nejnovější release k 2026-11-20). Cíl: nahrávat živé
streamy z kick.com do stejné knihovny jako Twitch – video, chat a metadata – bez rozbití
Twitche.

## 1. Analýza architektury (kde je Twitch „zadrátovaný“)

Ganymede = Go API (`cmd/server`) + Go worker (`cmd/worker`, River joby nad PostgreSQL)
+ Next.js frontend. Upstream už má rozhraní `internal/platform.Platform`, ale jedinou
implementací je `TwitchConnection` a všude se předává jako `PlatformTwitch`.

| Oblast | Soubor | Twitch závislost |
|---|---|---|
| Datový model | `ent/schema/vod.go` | `vods.platform` enum už existuje (twitch/youtube) |
| | `ent/schema/channel.go` | kanál **nemá** platformu; `name`, `display_name`, `ext_id` jsou globálně unikátní |
| Detekce live | `internal/live/live.go` `Check()` | Helix `GetLiveStreams` po 99 ID; filtr `vod.platform = twitch` |
| Start archivace | `internal/archive/archive.go` `ArchiveLivestream` | `PlatformTwitch.GetLiveStream`, `Platform: "twitch"` natvrdo |
| Nahrávání videa | `internal/exec/exec.go` `DownloadTwitchLiveVideo` | Twitch GQL playback token → usher m3u8 → **ffmpeg** (ne streamlink, ne yt-dlp) |
| Live chat | `internal/exec/twitch.go` | anonymní Twitch IRC |
| Konverze chatu | `internal/tasks/live_chat.go` | Twitch `GetChannel`/`GetVideos` + TwitchDownloaderCLI `chatupdate` (embed emotů) |
| Render chatu | `internal/exec/exec.go` | TwitchDownloaderCLI |
| Periodika | `internal/tasks/periodic/*`, `internal/live/{vod,clip}.go` | refresh kanálů, kapitoly, hlídání VOD/klipů – vše Twitch API |
| Přehrávání chatu | `internal/vod/vod.go` | fallback emoty/badge z Twitch + 7TV/BTTV/FFZ (Twitch ID) |
| Frontend | `PlatformDrawerContent.tsx` | přidání kanálu jen jménem (Twitch) |

### Co dělá „místní patch“ `ganymede_twitch_yt_dlp_git.patch`
Není místní. Je to **upstreamový** soubor (v `/opt/ganymede-main` v CT114 je bajtově
shodný s v5.0.3) a patchuje **yt-dlp** (Twitch extractor: GQL přes `requests`, persisted
query `PlaybackAccessToken`). V Dockerfile je jeho aplikace **zakomentovaná** – v buildu se
nepoužívá ani upstream, ani u nás. `/opt/ganymede-main` je jen starší (únor 2026) snapshot
upstreamu, běžící kontejner z něj buildovaný není. Pro Kick je patch irelevantní.

## 2. Návrh

Kick jako druhá implementace `platform.Platform` + směrování podle `channel.platform`.
Twitch cesty zůstávají beze změny chování.

- **`internal/platform/kick.go` – `KickConnection`** nad neoficiálním web API
  (stejné endpointy jako kick.com frontend a streamlink plugin):
  - `GET kick.com/api/v2/channels/{slug}` – kanál, avatar, livestream (titulek, kategorie,
    start, thumbnail), `playback_url` (IVS m3u8, **nepodepsaný**), `chatroom.id`.
  - `POST web.kick.com/api/v1/realtime/channels/{id}/chat/connection` – aktuální Pusher
    app key/cluster (Kick klíč rotuje; fallback na známý).
  - `GET kick.com/emotes/{slug}` – emote sety.
  - Retry s backoffem na 403/429/5xx (Cloudflare), browser-like hlavičky.
  - Operace, které Kick nemá (VOD/klipy/kategorie/bulk live), vrací prázdno nebo
    `ErrNotSupported`, aby následné joby degradovaly bez chyby.
- **Datový model** (aditivní, auto-migrace ent): `channels.platform` (enum, default
  `twitch` → existující kanály se samy doplní) a `channels.ext_login` (skutečný slug).
  Kick `ext_id` = `kick:<id>` (číselná ID Kicku a Twitche se překrývají, `ext_id` je unique).
  Kolize jména s Twitch kanálem → název `<slug>_kick`, display name `… (Kick)`; složka na
  disku se tím taky oddělí.
- **Detekce live**: Twitch dávkování beze změny (Kick kanály se přeskočí); Kick kanály
  se dotazují po jednom s 500 ms rozestupem. **Selhání Kick API nikdy nepřeruší Twitch
  check a nikdy neoznačí běžící nahrávání jako offline.**
- **Video**: Kick playlist jde do **stejné** ffmpeg/HLS pipeline (stall watchdog, recovery
  proxy, HLS/MP4 export) – Kick běží na Amazon IVS jako Twitch, varianty mají stejné názvy
  (`chunked` = source 1080p60, `720p60`, …), takže výběr kvality funguje beze změn.
  Streamlink není potřeba.
- **Chat**: vlastní Pusher websocket klient (`internal/exec/kick_chat.go`), zapisuje do
  sdíleného crash-safe live-chat formátu; `[emote:ID:name]` → text + pozice emotů; badge z
  `badges_v2` (s URL obrázku); repliky. Konverze sdíleným převodníkem, místo
  TwitchDownloader `chatupdate` vlastní `EmbedKickChatAssets` (base64 emoty + badge →
  archiv přehratelný offline). Synchronizace chatu kotví na start nahrávání.
  Render chatu do videa je pro Kick vypnutý (TwitchDownloader Kick neumí).
- **Twitch-only joby** (refresh kanálů, kapitoly VOD, hlídání VOD/klipů) Kick kanály
  přeskakují.
- **API/Frontend**: `POST /api/v1/archive/channel` má volitelné `platform: "kick"`;
  v Adminu → Kanály → přidat je přepínač Twitch/Kick a sloupec Platforma.

## 3. Rizika a omezení

| Riziko | Dopad | Mitigace |
|---|---|---|
| Cloudflare/změna neoficiálního API | Kick detekce/metadata selžou | retry+backoff; selhání izoluje jen Kick; z homelab IP i Go klienta dnes 200 |
| Rotace Pusher klíče | chat se nepřipojí | klíč se načítá dynamicky z Kick endpointu, fallback na známý |
| Kick VOD API chybí | žádné stahování Kick VOD/klipů, chat bez dohledání VOD | jen live; chat sync = start nahrávání (stejně jako Twitch fallback) |
| Sub-only / 18+ streamy | playlist může vyžadovat přihlášení | zatím nepodporováno (anonymní) |
| Render chatu | žádné chat-video pro Kick | přehrávání chatu v UI funguje (embedded emoty/badge) |
| Upgrade 5.0.0 → 5.0.3 | migrace DB | otestováno na kopii produkční DB v testovacím CT |

## 4. Nasazení

Vlastní image (`Dockerfile` beze změn) → `ganymede-kick:<tag>`, buildovaný v testovacím
CT 144 (`ganymede-kick-test`), přenesený do CT114 přes `docker save | docker load`.
V `/root/docker-compose.yml` jen změna `image:` (záloha compose + `pg_dump` před přepnutím;
přepínat jen když nic nenahrává). Rollback = vrátit `image: ghcr.io/zibbp/ganymede:5.0.0`
a obnovit DB dump (nové sloupce staré verzi nevadí, ent je ignoruje).

Přidání Kick kanálu: Admin → Channels → Add → **Kick** → slug (např. `xqc`), pak Admin →
Watched → přidat kanál s „Watch Live“.
