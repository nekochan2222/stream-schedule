// Node 18+。YouTube Data API から配信情報を取得し data/streams.json に保存する。
import fs from 'fs';

const KEY = process.env.YT_API_KEY;
if (!KEY) throw new Error('YT_API_KEY が未設定です');
const cfg = JSON.parse(fs.readFileSync('config.json', 'utf8'));
const OUT = 'data/streams.json';
const old = fs.existsSync(OUT) ? JSON.parse(fs.readFileSync(OUT, 'utf8')) : [];
const pages = old.length && !process.env.FULL ? 2 : cfg.fullPages || 20;

const api = async (path, q) => {
  const r = await fetch(`https://www.googleapis.com/youtube/v3/${path}?` + new URLSearchParams({ ...q, key: KEY }));
  if (!r.ok) throw new Error(await r.text());
  return r.json();
};

async function uploadIds(channelId) {
  const ids = [];
  let token;
  for (let i = 0; i < pages; i++) {
    const j = await api('playlistItems', {
      part: 'contentDetails', playlistId: 'UU' + channelId.slice(2), maxResults: 50,
      ...(token && { pageToken: token }),
    });
    ids.push(...j.items.map((x) => x.contentDetails.videoId));
    token = j.nextPageToken;
    if (!token) break;
  }
  return ids;
}

async function details(ids) {
  const out = [];
  for (let i = 0; i < ids.length; i += 50) {
    const j = await api('videos', { part: 'snippet,liveStreamingDetails', id: ids.slice(i, i + 50).join(',') });
    out.push(...j.items);
  }
  return out;
}

const gameOf = (title) => {
  for (const m of title.matchAll(/【([^】]+)】/g)) {
    if (!cfg.ignoreTags.includes(m[1])) return m[1];
  }
  return null;
};

const map = new Map(old.map((s) => [s.id, s]));
for (const ch of cfg.channels) {
  if (!ch.id && ch.handle) {
    const j = await api('channels', { part: 'id', forHandle: ch.handle });
    ch.id = j.items?.[0]?.id;
    if (!ch.id) throw new Error('ハンドルが見つかりません: ' + ch.handle);
  }
  const vids = await details(await uploadIds(ch.id));
  for (const v of vids) {
    const l = v.liveStreamingDetails;
    const start = l && (l.actualStartTime || l.scheduledStartTime);
    if (!start) continue;
    const text = v.snippet.title + '\n' + v.snippet.description;
    if (!ch.own && !cfg.keywords.some((k) => text.includes(k))) continue;
    map.set(v.id, {
      id: v.id,
      own: !!ch.own,
      official: !!ch.official,
      channel: v.snippet.channelTitle,
      title: v.snippet.title,
      game: gameOf(v.snippet.title),
      start,
      end: l.actualEndTime || null,
      status: v.snippet.liveBroadcastContent,
      thumb: (v.snippet.thumbnails.medium || v.snippet.thumbnails.default).url,
    });
  }
}

fs.mkdirSync('data', { recursive: true });
const all = [...map.values()].sort((a, b) => new Date(a.start) - new Date(b.start));
fs.writeFileSync(OUT, JSON.stringify(all));
console.log(`saved ${all.length} streams`);
