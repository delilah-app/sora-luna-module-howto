# How to create a Sora / Luna module

Short answer: **yes, there is an official guide.**

[Sora Modules Developer Specification](https://git.luna-app.eu/50n50/sources/src/branch/main/SORA_MODULES_GUIDE.md)

That spec is the contract for **Sora-style** hosts (Sora, Luna, Dartotsu, Anymex, Tsumi, Hiyoku, Mojuru, and others *when they implement this API*). **Luna (Kanzen)** can also load a source with a slightly different function set and extra runtime helpers. Do not mix the two contracts in one script.

Working Luna/Kanzen manga example (Churly’s Weebcentral, tweaked — fully functional again):

- [Weebcentral-Kanzen.json](https://raw.githubusercontent.com/JayGxnzalez/Weebcentral-Kanzen/refs/heads/main/Weebcentral-Kanzen.json)
- [Weebcentral-Kanzen.js](https://raw.githubusercontent.com/JayGxnzalez/Weebcentral-Kanzen/refs/heads/main/Weebcentral-Kanzen.js)

---

## 1. What a module is

Two files, hosted at stable raw URLs:

1. **Manifest** — `[module].json` (name, type, script URL, compat flags)
2. **Script** — one self-contained `.js` file (no imports)

You add the **json URL** as a source in the app. The app downloads the script from `scriptUrl` / `scriptURL`.

---

## 2. Runtime rules (Sora spec)

| Rule | Detail |
| --- | --- |
| Engine | Isolated JavaScriptCore or QuickJS (Android/iOS) |
| No DOM | No `document`, `window`, `DOMParser`, `XMLHttpRequest`, `LocalStorage`, `location` |
| No imports | No `require()`, `import`, or extra script loads. Unpackers go **inline** |
| Network | Browser `fetch` often fails (CORS). Use host `fetchv2` |
| Scope | Global `async function`s. Do **not** wrap in an IIFE |
| Errors | Never throw. `try/catch` every entry point; return empty fallbacks |

---

## 3. Manifest

Sora spec shape:

```json
{
  "sourceName": "Example Module",
  "iconUrl": "https://url.to/icon.png",
  "author": {
    "name": "Developer Name",
    "icon": "https://url.to/author-icon.png"
  },
  "version": "1.0.0",
  "language": "English",
  "baseUrl": "https://example.com",
  "searchBaseUrl": "https://example.com/search?q=",
  "scriptUrl": "https://raw.githubusercontent.com/.../module.js",
  "type": "anime",
  "streamType": "HLS",
  "asyncJS": true,
  "quality": "1080p",
  "downloadSupport": true,
  "softsub": true,
  "novel": true,
  "supportsSora": true,
  "supportsLuna": true,
  "supportsDartotsu": true,
  "supportsAnymex": true,
  "supportsTsumi": true,
  "supportsHiyoku": true,
  "supportsShirox": true,
  "supportsMojuru": true
}
```

`type`: `"anime"` | `"mangas"` | `"novels"`.

**Luna/Kanzen example** uses slightly different keys (`iconURL`, `scriptURL`) and only `supportsLuna`:

```json
{
  "sourceName": "Weebcentral",
  "iconURL": "https://favicons.statusgator.com/1bi5zotvuQiK08wI.png",
  "version": "1.0.2",
  "language": "English",
  "scriptURL": "https://raw.githubusercontent.com/JayGxnzalez/Weebcentral-Kanzen/refs/heads/main/Weebcentral-Kanzen.js",
  "author": {
    "name": "Jay",
    "iconURL": "https://avatars.githubusercontent.com/JayGxnzalez?s=400"
  },
  "type": "mangas",
  "supportsLuna": true
}
```

Match key spelling to the host you care about (`iconUrl` vs `iconURL`).

---

## 4. The three Sora types (required functions)

### Anime (`type: "anime"`)

Return **JSON strings** (`JSON.stringify`).

| Function | Returns |
| --- | --- |
| `searchResults(keyword)` | `[{ title, image, href }]` |
| `extractDetails(url)` | `[{ description, aliases, airdate }]` (one object in an array) |
| `extractEpisodes(url)` | `[{ href, number }]` |
| `extractStreamUrl(url)` | `{ streams: [{ title, streamUrl, headers? }], subtitles? }` |

### Manga (`type: "mangas"`)

Return **raw objects/arrays** (not stringified).

| Function | Returns |
| --- | --- |
| `searchResults(keyword, page?)` | `[{ id, title, imageURL }]` |
| `extractDetails(id)` | `{ description, tags: string[] }` |
| `extractChapters(urlOrId)` | chapters grouped by language code |
| `extractImages(chapterId)` | `string[]` (page image URLs) |

### Novels (`type: "novels"`)

JSON-stringify everything **except** `extractText` (raw HTML).

| Function | Returns |
| --- | --- |
| `searchResults(keyword)` | `[{ title, image, href }]` |
| `extractDetails(url)` | `[{ description, aliases, airdate }]` |
| `extractChapters(url)` | `[{ title, href, number }]` |
| `extractText(url)` | sanitized HTML string |

---

## 5. `fetchv2` + `soraFetch` (Sora)

Host injects `fetchv2(url, headers, method, body)`.

```js
async function soraFetch(url, options = { headers: {}, method: 'GET', body: null }) {
    const headers = options.headers || {};
    if (!headers["User-Agent"]) {
        headers["User-Agent"] = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36";
    }
    try {
        return await fetchv2(url, headers, options.method || 'GET', options.body || null);
    } catch (e) {
        try {
            return await fetch(url, options);
        } catch (error) {
            return null;
        }
    }
}
```

Then `const html = await response.text()`.

The Weebcentral Luna script calls `fetch` directly. That works **there**. For Sora-style hosts, start from `soraFetch` / `fetchv2`.

---

## 6. Parsing HTML

### Sora spec: no DOM

Use `indexOf` / `substring` / `split` or regex:

```js
const htmlText = await response.text();

const startIdx = htmlText.indexOf('id="chapterText"');
const chunk = htmlText.substring(startIdx, htmlText.indexOf('</div>', startIdx));

const regex = /<a href="([^"]+)".*?>\s*(.*?)\s*<\/a>/gi;
let match;
while ((match = regex.exec(htmlText)) !== null) {
    results.push({ href: match[1], title: match[2] });
}
```

Stream sites sometimes pack URLs (`eval(function(p,a,c,k,e,d)...)`). Put any unpacker **inline** in the same file. No extra scripts.

### Luna / Kanzen: htmlparser2 + cssSelect

The working Weebcentral module does **not** regex the page. It uses host-provided helpers:

```js
const dom = KanzenBundle.htmlparser2.parseDocument(text);
const articles = KanzenBundle.cssSelect.selectAll('article', dom);
```

That API exists in **Kanzen**. It is **not** part of the Sora spec. If you copy Weebcentral into a Sora-only host, `KanzenBundle` will be undefined.

---

## 7. Luna / Kanzen function names (from Weebcentral)

Do **not** implement `searchResults` / `extractDetails` / `extractChapters` / `extractImages` if you are writing a **Kanzen manga** source like Weebcentral. That example exports:

| Luna / Kanzen | Role |
| --- | --- |
| `searchContent(input, page = 0)` | Search → `[{ title, id, imageURL }]` |
| `getContentData(id)` | Details → `{ description, tags }` |
| `getChapters(id)` | Chapter list (example groups under `"eng"`) |
| `getChapterImages(params)` | Image URL array |

| Sora spec | Luna / Kanzen (Weebcentral) |
| --- | --- |
| `searchResults` | `searchContent` |
| `extractDetails` | `getContentData` |
| `extractChapters` | `getChapters` |
| `extractImages` | `getChapterImages` |

Same idea, **different entry points**. A host looking for `searchResults` will not call `searchContent`.

---

## 8. Minimal checklist to ship

1. Decide host: **Sora-style** (spec functions + `fetchv2` + no DOM) or **Luna/Kanzen** (Weebcentral-style names + `KanzenBundle` if you parse HTML that way).
2. Pick `type`: `anime` \| `mangas` \| `novels`.
3. Publish a **manifest** with `sourceName`, `type`, script URL, and the right compat flag (`supportsSora` / `supportsLuna`). Match key spelling to the host.
4. One **global** `.js` file. No IIFE, no `import`/`require`.
5. Implement **only** the functions for that host + type.
6. **anime / novels** → `JSON.stringify` (except `extractText` = raw HTML). **mangas** → raw objects.
7. Wrap every entry point in `try/catch`; return empty list / empty object on failure.
8. Test search → details → list → media (stream / images / chapter HTML) on a real title.
9. Host the two files at stable raw URLs and add the **json** URL as a source.

---

## 9. Links

- Official spec: https://git.luna-app.eu/50n50/sources/src/branch/main/SORA_MODULES_GUIDE.md
- Weebcentral-Kanzen manifest: https://raw.githubusercontent.com/JayGxnzalez/Weebcentral-Kanzen/refs/heads/main/Weebcentral-Kanzen.json
- Weebcentral-Kanzen script: https://raw.githubusercontent.com/JayGxnzalez/Weebcentral-Kanzen/refs/heads/main/Weebcentral-Kanzen.js

*This is a how-to, not a second spec. When the two disagree, the spec wins for Sora; the working Weebcentral files win for Luna/Kanzen manga.*
