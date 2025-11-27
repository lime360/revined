<script lang="ts">
    import Header from "$lib/components/Header.svelte";
    import { onMount } from "svelte";
    import { page } from "$app/stores";
    import { writable } from "svelte/store";

    const feed = writable([]);

    const currentRelay = $page.url.searchParams.get("relay") || "wss://relay.divine.video";

    function getURL(tags: string[][]) {
        const imetas = tags.filter(t => t[0] === "imeta");
        for (const im of imetas) {
            const i = im.indexOf("url");
            if (i !== -1 && im[i+1]) return im[i+1];
        }

        const url = tags.find(t => t[0] === "url")?.[1];
        if (url) return url;

        const r = tags.find(t => t[0] === "r")?.[1];
        if (r?.startsWith("http")) return r;

        const e = tags.find(t => t[0] === "e")?.[1];
        if (e?.startsWith("http")) return e;

        const i = tags.find(t => t[0] === "i")?.[1];
        if (i?.startsWith("http")) return i;

        for (const t of tags) {
            const found = t.find(v => typeof v === "string" && v.startsWith("http"));
            if (found) return found;
        }

        const match = /(https?:\/\/\S+\.mp4)/.exec(JSON.stringify(tags));
        return match?.[1];
    }

    function handleEvent(e) {
        if (e.kind !== 34236) return;

        const tags = e.tags || [];
        const d = tags.find(t => t[0] === "d")?.[1];
        const title = tags.find(t => t[0] === "title")?.[1] || "Untitled";
        const published = tags.find(t => t[0] === "published_at")?.[1];
        const videoUrl = getURL(tags);

        feed.update(v => [
            {
                id: d,
                title,
                videoUrl,
                published,
                content: e.content,
                pubkey: e.pubkey
            },
            ...v
        ]);
    }

    onMount(() => {
        const con = new WebSocket(currentRelay);

        con.onopen = () => con.send(JSON.stringify(["REQ", "reined", { kinds: [34236] }]));

        con.onmessage = (msg) => {
            const data = JSON.parse(msg.data);
            if (data[0] === "EVENT") handleEvent(data[2]);
        };

        return () => con.close();
    });
</script>

<Header />

<div class="container">
    <div class="fixed-grid has-3-cols">
        <div class="grid">
            {#each $feed as v}
                <div class="cell p-8 m-5">
                    {#if v.videoUrl}
                        <video
                            src={v.videoUrl}
                            autoplay
                            loop
                            playsinline
                            muted
                            controls
                        ></video>
                    {/if}

                    <p>{v.title}</p>
                    <small>
                        {#if v.published}
                            {new Date(parseInt(v.published) * 1000).toLocaleString()}
                        {/if}
                    </small>
                </div>
            {/each}
        </div>
    </div>
</div>
