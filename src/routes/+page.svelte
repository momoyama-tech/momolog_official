<script lang="ts">
import { onMount } from 'svelte';
  import { collection, onSnapshot, query, where, type Unsubscribe } from 'firebase/firestore';
  import { db } from '$lib/firebase';

  type GroupDoc = {
    id: string;
    name: string;
    description: string;
    youtube: {
      channelId: string;
      channelTitle: string;
      connected: boolean;
    };
  };

  type VideoDoc = {
    id: string;
    groupId: string;
    groupName: string;
    title: string;
    description: string;
    youtubeUrl: string;
    youtubeVideoId: string;
    youtubeThumbnailUrl: string;
    youtubeChannelId: string;
    youtubeChannelTitle: string;
    status: string;
    createdAt?: unknown;
  };

  type GroupSection = {
    group: GroupDoc;
    videos: VideoDoc[];
  };

  let groups: GroupDoc[] = [];
  let videos: VideoDoc[] = [];
  let loading = true;
  let error = '';
  let debugInfo = '';
  const allGroupsOptionId = '__all__';
  let selectedGroupId = '';
  let currentPage = 1;
  const videosPerPage = 10;
  let sortOrder: 'newest' | 'oldest' = 'newest';

  const fallbackGroupName = '未登録団体';

  const normalizeTimestamp = (value: unknown): number => {
    if (typeof value === 'number') return value;
    if (value instanceof Date) return value.getTime();
    if (typeof value === 'object' && value !== null && 'toMillis' in value) {
      const candidate = value as { toMillis?: unknown };
      if (typeof candidate.toMillis === 'function') {
        return candidate.toMillis();
      }
    }
    if (typeof value === 'object' && value !== null && 'seconds' in value) {
      const ts = value as { seconds?: unknown; nanoseconds?: unknown };
      const seconds = typeof ts.seconds === 'number' ? ts.seconds : 0;
      const nanoseconds = typeof ts.nanoseconds === 'number' ? ts.nanoseconds : 0;
      return seconds * 1000 + Math.floor(nanoseconds / 1_000_000);
    }
    return 0;
  };

  const sortVideos = (a: VideoDoc, b: VideoDoc): number => {
    const timeDiff = normalizeTimestamp(b.createdAt) - normalizeTimestamp(a.createdAt);
    if (timeDiff !== 0) return timeDiff;
    return a.title.localeCompare(b.title, 'ja');
  };

  const formatVideoDate = (value: unknown): string => {
    const millis = normalizeTimestamp(value);
    if (!millis) return '公開日時未設定';
    return new Date(millis).toLocaleString('ja-JP', {
      year: 'numeric',
      month: '2-digit',
      day: '2-digit',
      hour: '2-digit',
      minute: '2-digit'
    });
  };

  const normalizeKey = (value: string): string => value.normalize('NFKC').trim().toLocaleLowerCase('ja-JP');

  $: effectiveGroups = (() => {
    if (groups.length > 0) return groups;

    const fallback = new Map<string, GroupDoc>();
    for (const video of videos) {
      const key = video.groupId.trim() || normalizeKey(video.groupName) || `video-${video.id}`;
      if (fallback.has(key)) continue;
      fallback.set(key, {
        id: video.groupId.trim() || key,
        name: video.groupName.trim() || fallbackGroupName,
        description: '',
        youtube: {
          channelId: video.youtubeChannelId,
          channelTitle: video.youtubeChannelTitle,
          connected: false
        }
      });
    }

    return Array.from(fallback.values());
  })();

  $: groupSections = (() => {
    const sections: GroupSection[] = effectiveGroups
      .slice()
      .sort((a, b) => a.name.localeCompare(b.name, 'ja'))
      .map((group) => ({
        group,
        videos: videos
          .filter((video) => {
            const groupNameKey = normalizeKey(group.name);
            const videoGroupNameKey = normalizeKey(video.groupName);
            const byGroupId = video.groupId.trim() !== '' && video.groupId.trim() === group.id.trim();
            const byGroupName = groupNameKey !== '' && videoGroupNameKey !== '' && videoGroupNameKey === groupNameKey;
            return byGroupId || byGroupName;
          })
          .slice()
          .sort(sortVideos)
      }));

    return sections;
  })();

  $: if (
    groupSections.length > 0 &&
    selectedGroupId !== allGroupsOptionId &&
    !groupSections.some((section) => section.group.id === selectedGroupId)
  ) {
    selectedGroupId = allGroupsOptionId;
  }

  $: selectedGroupSection = groupSections.find((section) => section.group.id === selectedGroupId);
  $: currentGroupSection = selectedGroupSection ?? groupSections[0];
  $: selectedVideosBase = selectedGroupId === allGroupsOptionId ? videos : (selectedGroupSection?.videos ?? []);
  $: sortedSelectedVideos = (() => {
    const allVideos = [...selectedVideosBase];
    if (sortOrder === 'newest') {
      return allVideos.sort(sortVideos);
    }
    return allVideos.sort((a, b) => sortVideos(b, a));
  })();
  $: totalPages = Math.max(1, Math.ceil(sortedSelectedVideos.length / videosPerPage));
  $: if (currentPage > totalPages) currentPage = totalPages;
  $: paginatedVideos = (() => {
    const start = (currentPage - 1) * videosPerPage;
    return sortedSelectedVideos.slice(start, start + videosPerPage);
  })();

  onMount(() => {
    const unsubs: Unsubscribe[] = [];
    let groupsReady = false;
    let videosReady = false;

    const finishLoadingIfReady = () => {
      if (groupsReady && videosReady) {
        loading = false;
      }
    };

    try {
      const groupsQuery = query(collection(db, 'groups'));
      const videosQuery = query(collection(db, 'videos'), where('status', '==', 'published'));

      unsubs.push(
        onSnapshot(
          groupsQuery,
          (snapshot) => {
            groups = snapshot.docs.map((doc) => {
              const data = doc.data() as Record<string, unknown>;
              const youtube = (data.youtube ?? {}) as Record<string, unknown>;
              return {
                id: doc.id,
                name: String(data.name ?? ''),
                description: String(data.description ?? ''),
                youtube: {
                  channelId: String(youtube.channelId ?? ''),
                  channelTitle: String(youtube.channelTitle ?? ''),
                  connected: Boolean(youtube.connected)
                }
              };
            });

            groupsReady = true;
            finishLoadingIfReady();
          },
          (err) => {
            const code = typeof err === 'object' && err && 'code' in err ? String(err.code) : 'unknown';
            const message = typeof err === 'object' && err && 'message' in err ? String(err.message) : '';
            error = '団体情報の取得に失敗したため、動画データから団体名を補完表示しています。';
            debugInfo = `groups error: ${code}${message ? ` / ${message}` : ''}`;
            groupsReady = true;
            finishLoadingIfReady();
          }
        )
      );

      unsubs.push(
        onSnapshot(
          videosQuery,
          (snapshot) => {
            videos = snapshot.docs.flatMap((doc) => {
                const data = doc.data() as Record<string, unknown>;
                const rawStatus = String(data.status ?? '');
                const normalizedStatus = rawStatus.trim().toLowerCase();
                if (normalizedStatus !== 'published') {
                  return [];
                }

                return [
                  {
                    id: doc.id,
                    groupId: String(data.groupId ?? ''),
                    groupName: String(data.groupName ?? ''),
                    title: String(data.title ?? ''),
                    description: String(data.description ?? ''),
                    youtubeUrl: String(data.youtubeUrl ?? ''),
                    youtubeVideoId: String(data.youtubeVideoId ?? ''),
                    youtubeThumbnailUrl: String(data.youtubeThumbnailUrl ?? ''),
                    youtubeChannelId: String(data.youtubeChannelId ?? ''),
                    youtubeChannelTitle: String(data.youtubeChannelTitle ?? ''),
                    status: rawStatus,
                    createdAt: data.createdAt
                  }
                ];
              });

            videosReady = true;
            finishLoadingIfReady();
          },
          (err) => {
            const code = typeof err === 'object' && err && 'code' in err ? String(err.code) : 'unknown';
            const message = typeof err === 'object' && err && 'message' in err ? String(err.message) : '';
            error = '動画情報の取得に失敗しました。';
            debugInfo = `videos error: ${code}${message ? ` / ${message}` : ''}`;
            loading = false;
          }
        )
      );
    } catch {
      error = 'Firestore の設定が不正です。.env の公開変数を確認してください。';
      loading = false;
    }

    return () => {
      for (const unsubscribe of unsubs) {
        unsubscribe();
      }
    };
  });
</script>

<svelte:head>
  <title>団体別動画一覧 | momolog official</title>
  <meta name="description" content="momolog official の団体別動画一覧" />
</svelte:head>

<div class="page-shell">
  <div class="ambient ambient-left" aria-hidden="true"></div>
  <div class="ambient ambient-right" aria-hidden="true"></div>
  <div class="noise" aria-hidden="true"></div>

  <main class="container">
    <section class="hero">
      <p class="hero-kicker">MOMOLOG OFFICIAL</p>
      <h1>団体別動画アーカイブ</h1>
      <p class="hero-sub">旬の動画を、団体ごとに。</p>
    </section>

    {#if loading}
      <p class="notice">団体と動画を読み込み中です...</p>
    {:else}
      {#if error}
        <p class="notice is-error">{error}</p>
        {#if debugInfo}
          <p class="notice is-error is-debug">{debugInfo}</p>
        {/if}
      {/if}

      {#if groupSections.length === 0}
        <p class="notice">現在、公開中の団体・動画はありません。（groups: {groups.length} / videos: {videos.length}）</p>
      {:else}
        <div class="group-stack">
          <div class="controls">
            <div class="group-selector">
              <label for="group-select">部活動を選択</label>
              <select
                id="group-select"
                bind:value={selectedGroupId}
                on:change={() => {
                  currentPage = 1;
                }}
              >
                <option value={allGroupsOptionId}>すべての動画</option>
                {#each groupSections as section (section.group.id)}
                  <option value={section.group.id}>{section.group.name || fallbackGroupName}</option>
                {/each}
              </select>
            </div>
            <div class="group-selector sort-selector">
              <label for="sort-select">並び順</label>
              <select
                id="sort-select"
                bind:value={sortOrder}
                on:change={() => {
                  currentPage = 1;
                }}
              >
                <option value="newest">新着順</option>
                <option value="oldest">古い順</option>
              </select>
            </div>
          </div>

          {#if selectedGroupId === allGroupsOptionId || currentGroupSection}
            <section class="group-block" role="tabpanel">
              <header class="group-head">
                <div>
                  <h2>
                    {#if selectedGroupId === allGroupsOptionId}
                      すべての動画
                    {:else}
                      {currentGroupSection.group.name || fallbackGroupName}
                    {/if}
                  </h2>
                  <p class="group-description">
                    {#if selectedGroupId === allGroupsOptionId}
                      公開中の全動画を一覧表示しています。
                    {:else}
                      {currentGroupSection.group.description || '団体説明は未設定です。'}
                    {/if}
                  </p>
                </div>
                <div class="group-meta">
                  {#if selectedGroupId === allGroupsOptionId}
                    <p class="group-id">公開動画: {videos.length}件</p>
                  {:else}
                    <p class="group-id">ID: {currentGroupSection.group.id}</p>
                    <p class="channel-line">
                      {#if currentGroupSection.group.youtube.channelId}
                        <a href={`https://www.youtube.com/channel/${currentGroupSection.group.youtube.channelId}`} target="_blank" rel="noreferrer">
                          {currentGroupSection.group.youtube.channelTitle || 'YouTubeチャンネル'}
                        </a>
                      {:else if currentGroupSection.videos[0]?.youtubeChannelId}
                        <a href={`https://www.youtube.com/channel/${currentGroupSection.videos[0].youtubeChannelId}`} target="_blank" rel="noreferrer">
                          {currentGroupSection.videos[0].youtubeChannelTitle || 'YouTubeチャンネル'}
                        </a>
                      {:else}
                        YouTubeチャンネル未連携
                      {/if}
                      <span class="chip" data-connected={currentGroupSection.group.youtube.connected ? 'true' : 'false'}>
                        {currentGroupSection.group.youtube.connected ? 'connected' : 'not connected'}
                      </span>
                    </p>
                  {/if}
                </div>
              </header>

              {#if sortedSelectedVideos.length === 0}
                <p class="empty">公開中の動画はありません。</p>
              {:else}
                <ul class="video-feed">
                  {#each paginatedVideos as video (video.id)}
                    <li class="video-card">
                      <div class="video-media">
                        {#if video.youtubeVideoId}
                          <iframe
                            title={video.title}
                            src={`https://www.youtube.com/embed/${video.youtubeVideoId}`}
                            loading="lazy"
                            allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
                            allowfullscreen
                          ></iframe>
                        {:else if video.youtubeThumbnailUrl}
                          <img src={video.youtubeThumbnailUrl} alt={video.title} loading="lazy" />
                        {:else}
                          <div class="media-placeholder">サムネイルなし</div>
                        {/if}
                      </div>

                      <div class="video-body">
                        <p class="video-label">Campus Movie</p>
                        <h3>{video.title || 'タイトル未設定'}</h3>
                        <p class="video-meta">
                          <span>
                            {#if selectedGroupId === allGroupsOptionId}
                              {video.groupName || '団体名未設定'}
                            {:else}
                              {currentGroupSection.group.name || '団体名未設定'}
                            {/if}
                          </span>
                          <span>{video.youtubeChannelTitle || 'チャンネル未設定'}</span>
                          <span>{formatVideoDate(video.createdAt)}</span>
                        </p>
                        <p class="video-description">{video.description || '動画説明は未設定です。'}</p>

                        <div class="video-actions">
                          {#if video.youtubeUrl}
                            <a class="btn" href={video.youtubeUrl} target="_blank" rel="noreferrer">YouTubeで視聴</a>
                          {/if}
                          <span class="status">{video.status}</span>
                        </div>
                      </div>
                    </li>
                  {/each}
                </ul>
                <nav class="pagination" aria-label="動画ページネーション">
                  <button
                    type="button"
                    class="page-btn"
                    on:click={() => (currentPage = Math.max(1, currentPage - 1))}
                    disabled={currentPage === 1}
                  >
                    前へ
                  </button>
                  <span class="page-indicator">{currentPage} / {totalPages}</span>
                  <button
                    type="button"
                    class="page-btn"
                    on:click={() => (currentPage = Math.min(totalPages, currentPage + 1))}
                    disabled={currentPage === totalPages}
                  >
                    次へ
                  </button>
                </nav>
              {/if}
            </section>
          {/if}
        </div>
      {/if}
    {/if}
  </main>
</div>

<style>
  :global(body) {
    margin: 0;
    color: #f3f7fc;
    font-family: 'Yu Gothic UI', 'Meiryo UI', sans-serif;
    background: #111722;
  }

  .page-shell {
    position: relative;
    min-height: 100vh;
    padding: 24px 14px 80px;
    overflow: hidden;
    background: linear-gradient(150deg, #111722 0%, #1a2436 54%, #111926 100%);
    display: flex;
    justify-content: center;
  }

  .ambient {
    position: absolute;
    width: 520px;
    height: 520px;
    border-radius: 50%;
    filter: blur(46px);
    opacity: 0.42;
    pointer-events: none;
  }

  .ambient-left {
    left: -180px;
    top: -140px;
    background: radial-gradient(circle, rgba(173, 37, 90, 0.74), rgba(173, 37, 90, 0));
  }

  .ambient-right {
    right: -180px;
    top: 120px;
    background: radial-gradient(circle, rgba(44, 75, 116, 0.78), rgba(44, 75, 116, 0));
  }

  .noise {
    position: absolute;
    inset: 0;
    pointer-events: none;
    opacity: 0.14;
    background-image: linear-gradient(transparent 96%, rgba(255, 255, 255, 0.08) 100%);
    background-size: 100% 4px;
  }

  .container {
    --ui-scale: 0.65;
    width: 100%;
    max-width: 1260px;
    margin: 0 auto;
    position: relative;
    z-index: 1;
    flex: 0 0 auto;
    zoom: var(--ui-scale);
  }

  .hero {
    padding: 28px 22px;
    border-radius: 24px;
    margin-bottom: 20px;
    border: 1px solid rgba(189, 62, 105, 0.42);
    background:
      linear-gradient(130deg, rgba(26, 36, 53, 0.94), rgba(22, 31, 46, 0.92)),
      linear-gradient(180deg, rgba(189, 62, 105, 0.2), rgba(189, 62, 105, 0));
    box-shadow: 0 24px 50px rgba(0, 0, 0, 0.45);
  }

  .hero-kicker {
    margin: 0;
    font-size: 0.72rem;
    letter-spacing: 0.18em;
    font-weight: 700;
    color: #efb3c6;
    text-transform: uppercase;
  }

  h1 {
    margin: 10px 0 4px;
    font-size: clamp(2.1rem, 5vw, 3.4rem);
    line-height: 1.04;
    letter-spacing: 0.01em;
    color: #f5f8fc;
  }

  .hero-sub {
    margin: 0;
    font-size: 0.92rem;
    color: #b7c7d9;
  }

  .notice {
    margin: 0;
    border-radius: 14px;
    padding: 12px 14px;
    background: rgba(30, 41, 57, 0.9);
    border: 1px solid rgba(89, 111, 132, 0.46);
    color: #e4edf7;
    box-shadow: 0 10px 24px rgba(0, 0, 0, 0.28);
  }

  .notice.is-error {
    border-color: rgba(230, 106, 140, 0.66);
    background: rgba(57, 19, 33, 0.92);
    color: #ffd7e2;
  }

  .notice.is-debug {
    margin-top: 8px;
    font-size: 0.8rem;
    white-space: pre-wrap;
    word-break: break-word;
  }

  .group-stack {
    margin-top: 14px;
    display: grid;
    gap: 20px;
  }

  .group-selector {
    display: grid;
    gap: 6px;
    max-width: 420px;
  }

  .controls {
    display: flex;
    gap: 10px;
    flex-wrap: wrap;
    align-items: end;
  }

  .sort-selector {
    max-width: 220px;
  }

  .group-selector label {
    font-size: 0.78rem;
    color: #d2e1ee;
    font-weight: 700;
  }

  .group-selector select {
    appearance: none;
    width: 100%;
    border: 1px solid rgba(106, 128, 146, 0.65);
    background: rgba(24, 35, 48, 0.9);
    color: #edf4fb;
    border-radius: 12px;
    padding: 10px 12px;
    font-size: 0.88rem;
    font-weight: 700;
  }

  .group-block {
    border-radius: 24px;
    border: 1px solid rgba(84, 106, 126, 0.46);
    background:
      linear-gradient(180deg, rgba(27, 38, 55, 0.92) 0%, rgba(20, 30, 43, 0.95) 100%),
      repeating-linear-gradient(45deg, rgba(255, 255, 255, 0.03), rgba(255, 255, 255, 0.03) 10px, rgba(255, 255, 255, 0.01) 10px, rgba(255, 255, 255, 0.01) 20px);
    padding: 18px;
    box-shadow: 0 20px 40px rgba(0, 0, 0, 0.34);
  }

  .group-head {
    display: flex;
    justify-content: space-between;
    gap: 14px;
    flex-wrap: wrap;
    border-bottom: 1px solid rgba(83, 104, 121, 0.45);
    padding-bottom: 12px;
  }

  h2 {
    margin: 0;
    font-size: 1.35rem;
    color: #f3f7fc;
    letter-spacing: 0.01em;
  }

  .group-description {
    margin: 8px 0 0;
    font-size: 0.89rem;
    color: #d2deea;
    white-space: pre-wrap;
  }

  .group-meta {
    text-align: right;
    min-width: 280px;
  }

  .group-id {
    margin: 0;
    font-size: 0.76rem;
    color: #b1c4d5;
    font-weight: 700;
  }

  .channel-line {
    margin: 8px 0 0;
    display: inline-flex;
    gap: 8px;
    align-items: center;
    flex-wrap: wrap;
    justify-content: flex-end;
    color: #d5e4f2;
  }

  .channel-line a {
    color: #f2b4c8;
    font-weight: 700;
  }

  .chip {
    border-radius: 999px;
    padding: 3px 8px;
    font-size: 0.72rem;
    color: #cbd8e4;
    background: rgba(96, 117, 133, 0.35);
    text-transform: lowercase;
  }

  .chip[data-connected='true'] {
    color: #e0fbef;
    background: rgba(121, 175, 151, 0.34);
  }

  .empty {
    margin: 12px 0 0;
    color: #bdcede;
    font-size: 0.88rem;
  }

  .video-feed {
    list-style: none;
    margin: 14px 0 0;
    padding: 0;
    display: grid;
    gap: 16px;
  }

  .pagination {
    margin-top: 14px;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 10px;
  }

  .page-btn {
    border: 1px solid rgba(106, 128, 146, 0.55);
    background: rgba(24, 35, 48, 0.88);
    color: #d6e3ef;
    border-radius: 998px;
    padding: 6px 12px;
    font-size: 0.78rem;
    font-weight: 700;
    cursor: pointer;
  }

  .page-btn:disabled {
    opacity: 0.45;
    cursor: not-allowed;
  }

  .page-indicator {
    font-size: 0.78rem;
    color: #c3d3e3;
    min-width: 64px;
    text-align: center;
  }

  .video-card {
    width: 100%;
    border-radius: 18px;
    border: 1px solid rgba(82, 104, 123, 0.47);
    background: rgba(18, 27, 40, 0.88);
    display: grid;
    gap: 12px;
    padding: 12px;
  }

  .video-media {
    border-radius: 12px;
    overflow: hidden;
    border: 1px solid rgba(85, 108, 127, 0.5);
    background: #0f1724;
  }

  .video-media iframe,
  .video-media img,
  .media-placeholder {
    width: 100%;
    aspect-ratio: 16 / 9;
    min-height: 280px;
    display: block;
    border: 0;
  }

  .media-placeholder {
    display: grid;
    place-items: center;
    color: #bdd0e3;
    font-size: 0.86rem;
  }

  .video-body {
    padding: 4px 2px 2px;
  }

  .video-label {
    margin: 0 0 6px;
    font-size: 0.7rem;
    color: #efb3c6;
    font-weight: 700;
    letter-spacing: 0.1em;
    text-transform: uppercase;
  }

  h3 {
    margin: 0;
    color: #f4f8fc;
    font-size: 1.25rem;
    line-height: 1.35;
  }

  .video-meta {
    margin: 8px 0 0;
    display: flex;
    flex-wrap: wrap;
    gap: 10px;
    color: #c1d4e4;
    font-size: 0.82rem;
  }

  .video-description {
    margin: 10px 0 0;
    font-size: 0.9rem;
    line-height: 1.5;
    color: #dae5f0;
    white-space: pre-wrap;
    display: -webkit-box;
    -webkit-line-clamp: 3;
    line-clamp: 3;
    -webkit-box-orient: vertical;
    overflow: hidden;
  }

  .video-actions {
    margin-top: 12px;
    display: flex;
    align-items: center;
    flex-wrap: wrap;
    gap: 10px;
  }

  .btn {
    display: inline-block;
    text-decoration: none;
    border-radius: 999px;
    padding: 9px 15px;
    font-size: 0.8rem;
    font-weight: 700;
    color: #fff7fb;
    background: linear-gradient(90deg, #a72b59, #cb4b74);
    box-shadow: 0 10px 18px rgba(167, 43, 89, 0.36);
    transition: transform 0.18s ease, filter 0.18s ease;
  }

  .btn:hover {
    transform: translateY(-1px);
    filter: brightness(1.08);
  }

  .status {
    border-radius: 999px;
    border: 1px solid rgba(98, 122, 141, 0.52);
    padding: 5px 10px;
    font-size: 0.72rem;
    color: #d2e1ee;
    text-transform: uppercase;
    letter-spacing: 0.05em;
  }

  @media (min-width: 880px) {
    .page-shell {
      padding-top: 35px;
    }

    .hero {
      padding: 34px 28px;
    }

    .video-card {
      grid-template-columns: minmax(440px, 58%) 1fr;
      align-items: start;
      gap: 16px;
      padding: 14px;
    }

    .video-media iframe,
    .video-media img,
    .media-placeholder {
      min-height: 320px;
    }
  }
</style>

