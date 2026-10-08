You can automate this directly inside **Firefox** using a custom JavaScript snippet run in the Browser Developer Console.

Since you are already logged into Mahara-Tech, running the script in your browser session allows it to automatically use your session cookies, fetch each course page from the sidebar index (`#courseindex`), scan both iframes and hidden H5P JSON configuration payloads for YouTube links, and download the results as a JSON file.

### How to Run the Script in Firefox

1. Open your course page in Firefox (`[https://maharatech.gov.eg/mod/hvp/view.php?id=](https://maharatech.gov.eg/mod/hvp/view.php?id=)...`).
2. Open the **Web Developer Tools** by pressing `F12` (or right-clicking anywhere, selecting **Inspect**, and clicking the **Console** tab).
3. Copy the script below, paste it into the console, and press **Enter**.

```javascript
(async function extractYouTubeVideos() {
    // ---- Settings ----
    const MODULE_PATTERN = /\/mod\/(hvp|page|book|lesson|url)\//; // activity types to scan
    const DELAY_MS = 400;                                          // pause between requests

    // H5P stores its content as JSON with escaped slashes (https:\\/\\/youtube.com\\/...),
    // so we un-escape the text before searching it.
    const normalize = (s) => s
        .replace(/\\+u002f/gi, '/')
        .replace(/\\+\//g, '/')
        .replace(/\\+u0026/gi, '&')
        .replace(/&amp;/g, '&')
        .replace(/&#0?38;/g, '&');

    const YT_REGEX = /(?:youtube(?:-nocookie)?\.com\/(?:watch\?(?:[^"'\s<>]*?[?&])?v=|embed\/|v\/|shorts\/|live\/)|youtu\.be\/)([A-Za-z0-9_-]{11})/g;

    const findYouTubeIds = (rawText) => {
        const ids = new Set();
        for (const m of normalize(rawText).matchAll(YT_REGEX)) ids.add(m[1]);
        return ids;
    };

    const fetchText = async (url) => {
        const res = await fetch(url, { credentials: 'same-origin' });
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.text();
    };

    const sleep = (ms) => new Promise(r => setTimeout(r, ms));

    // 1. Collect activity pages from the course index sidebar + the "Jump to..." dropdown
    const seen = new Set();
    const pages = [];
    document.querySelectorAll('#courseindex a.courseindex-link, #jump-to-activity option').forEach(el => {
        const href = el.getAttribute('href') || el.value;
        if (!href || !MODULE_PATTERN.test(href)) return;
        const url = new URL(href, location.origin);
        const id = url.searchParams.get('id');
        if (!id) return;
        const clean = `${url.origin}${url.pathname}?id=${id}`;
        if (seen.has(clean)) return;
        seen.add(clean);
        pages.push({
            title: el.textContent.trim(),
            url: clean,
            id,
            isHvp: url.pathname.includes('/mod/hvp/')
        });
    });

    console.log(`📁 Found ${pages.length} activity pages to scan.`);
    if (!pages.length) {
        console.warn('No activity links found. Open the course index drawer and run the script again.');
        return;
    }

    const results = [];
    const missing = [];

    // 2. Scan each page
    for (let i = 0; i < pages.length; i++) {
        const page = pages[i];
        console.log(`[${i + 1}/${pages.length}] ${page.title}`);
        try {
            let html = await fetchText(page.url);
            let ids = findYouTubeIds(html);
            let source = 'view page';

            // Fallback for H5P: the embed page also carries the content settings
            if (!ids.size && page.isHvp) {
                try {
                    const embedHtml = await fetchText(`${location.origin}/mod/hvp/embed.php?id=${page.id}`);
                    ids = findYouTubeIds(embedHtml);
                    if (ids.size) source = 'embed page';
                } catch (e) { /* embed.php may not be available; ignore */ }
            }

            if (ids.size) {
                results.push({
                    pageTitle: page.title,
                    pageUrl: page.url,
                    foundIn: source,
                    youtubeLinks: [...ids].map(id => `https://www.youtube.com/watch?v=${id}`)
                });
                console.log(`   ✅ ${ids.size} link(s)`);
            } else {
                missing.push({
                    pageTitle: page.title,
                    pageUrl: page.url,
                    hasH5PData: /H5PIntegration/.test(html)
                });
                console.log('   ➖ no YouTube link found');
            }
        } catch (err) {
            missing.push({ pageTitle: page.title, pageUrl: page.url, error: String(err) });
            console.error(`   ❌ ${err}`);
        }
        await sleep(DELAY_MS);
    }

    console.log(`\n🎉 Done. ${results.length} pages with YouTube links, ${missing.length} without.`);
    console.table(results.map(r => ({ title: r.pageTitle, links: r.youtubeLinks.join(' , ') })));
    if (missing.length) {
        console.log('Pages with no YouTube link (check hasH5PData / error):');
        console.table(missing);
    }

    // 3. Download as JSON (Blob works more reliably than data: URIs in Firefox)
    const output = { extractedAt: new Date().toISOString(), results, missing };
    const blob = new Blob([JSON.stringify(output, null, 2)], { type: 'application/json' });
    const a = document.createElement('a');
    a.href = URL.createObjectURL(blob);
    a.download = 'maharatech_youtube_links.json';
    document.body.appendChild(a);
    a.click();
    a.remove();
    setTimeout(() => URL.revokeObjectURL(a.href), 5000);
    console.log("💾 Saved 'maharatech_youtube_links.json'");
})();

```

### What this script does:

1. **Scrapes the Sidebar:** Automatically queries all sidebar links (`#courseindex a.courseindex-link`), ensuring it picks up every H5P lesson or page module regardless of whether it's collapsed or expanded.
2. **Bypasses Authentication Walls:** Uses `fetch(..., { credentials: 'include' })` so your active Firefox login session is passed along with each sub-page request.
3. **Deep Content Parsing:** Because Mahara-Tech uses **H5P** interactive content (where YouTube links are often embedded inside JSON config strings rather than direct DOM iframes), the script uses a robust Regular Expression to extract video IDs from both iframes and script tags.
4. **Export:** Once finished, it prints a clean summary table to your console and automatically downloads a file named `maharatech_youtube_links.json` containing all lesson titles, page URLs, and associated YouTube links.
