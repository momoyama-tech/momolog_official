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

<div class="page-bg">
  <div class="wave wave-top" aria-hidden="true"></div>
  <div class="wave wave-bottom" aria-hidden="true"></div>

  <main class="stage">
    <section class="hero-panel">
      <p class="eyebrow">MOMOLOG OFFICIAL</p>
      <h1>団体別動画一覧</h1>
      <p class="collection-name">Firestore: /groups + /videos (status = published)</p>
    </section>

    {#if loading}
      <p class="message-card">団体と動画を読み込み中です...</p>
    {:else}
      {#if error}
        <p class="message-card is-error">{error}</p>
        {#if debugInfo}
          <p class="message-card is-error debug">{debugInfo}</p>
        {/if}
      {/if}
      {#if groupSections.length === 0}
        <p class="message-card">現在、公開中の団体・動画はありません。（groups: {groups.length} / videos: {videos.length}）</p>
      {:else}
      <div class="group-sections">
        {#each groupSections as section (section.group.id)}
          <section class="group-card">
            <div class="card-cap"></div>
            <h2>{section.group.name || fallbackGroupName}</h2>
            <p class="group-id">ID: {section.group.id}</p>
            <p class="group-description">{section.group.description || '団体説明は未設定です。'}</p>
            <p class="channel">
              {#if section.group.youtube.channelId}
                <a
                  href={`https://www.youtube.com/channel/${section.group.youtube.channelId}`}
                  target="_blank"
                  rel="noreferrer"
                >
                  {section.group.youtube.channelTitle || 'YouTubeチャンネル'}
                </a>
              {:else if section.videos[0]?.youtubeChannelId}
                <a
                  href={`https://www.youtube.com/channel/${section.videos[0].youtubeChannelId}`}
                  target="_blank"
                  rel="noreferrer"
                >
                  {section.videos[0].youtubeChannelTitle || 'YouTubeチャンネル'}
                </a>
              {:else}
                YouTubeチャンネル未連携
              {/if}
              <span class="channel-status" data-connected={section.group.youtube.connected ? 'true' : 'false'}>
                {section.group.youtube.connected ? 'connected' : 'not connected'}
              </span>
            </p>

            {#if section.videos.length === 0}
              <p class="empty-text">公開中の動画はありません。</p>
            {:else}
              <ul class="video-list">
                {#each section.videos as video (video.id)}
                  <li class="video-item">
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
                    <div class="video-info">
                      <h3>{video.title || 'タイトル未設定'}</h3>
                      <p>{video.description || '動画説明は未設定です。'}</p>
                      {#if video.youtubeUrl}
                        <a href={video.youtubeUrl} target="_blank" rel="noreferrer">YouTubeで開く</a>
                      {/if}
                    </div>
                  </li>
                {/each}
              </ul>
            {/if}
          </section>
        {/each}
      </div>
      {/if}
    {/if}
  </main>
</div>

<style>
  :global(body) {
    margin: 0;
    font-family: 'Yu Gothic UI', 'Meiryo UI', sans-serif;
  }

  .page-bg {
    min-height: 100vh;
    padding: 32px 16px 64px;
    position: relative;
    overflow: hidden;
    background:
      radial-gradient(circle at 10% 12%, rgba(255, 255, 255, 0.92) 0%, rgba(255, 255, 255, 0) 34%),
      radial-gradient(circle at 90% 18%, rgba(209, 246, 255, 0.72) 0%, rgba(255, 255, 255, 0) 36%),
      linear-gradient(160deg, #e8f8ff 0%, #f2fff9 48%, #f4f7ff 100%);
  }

  .wave {
    position: absolute;
    left: -6%;
    width: 112%;
    height: 220px;
    border-radius: 50%;
    pointer-events: none;
    opacity: 0.45;
    filter: blur(1px);
  }

  .wave-top {
    top: -140px;
    background: linear-gradient(90deg, rgba(132, 215, 255, 0.5), rgba(130, 241, 214, 0.45));
  }

  .wave-bottom {
    bottom: -150px;
    background: linear-gradient(90deg, rgba(130, 241, 214, 0.4), rgba(175, 210, 255, 0.45));
  }

  .stage {
    max-width: 1080px;
    margin: 0 auto;
    position: relative;
    z-index: 1;
  }

  .hero-panel {
    border-radius: 22px;
    padding: 24px 18px;
    margin-bottom: 18px;
    background: rgba(255, 255, 255, 0.78);
    border: 1px solid rgba(151, 225, 237, 0.9);
    box-shadow:
      0 18px 38px rgba(37, 106, 150, 0.14),
      inset 0 0 0 1px rgba(255, 255, 255, 0.7);
    backdrop-filter: blur(5px);
  }

  .eyebrow {
    margin: 0;
    font-size: 0.72rem;
    letter-spacing: 0.16em;
    font-weight: 700;
    color: #2c7ea5;
  }

  h1 {
    margin: 10px 0 6px;
    font-size: clamp(2rem, 4vw, 3.1rem);
    line-height: 1.06;
    color: #1f5e83;
    letter-spacing: 0.01em;
  }

  .collection-name {
    margin: 0;
    color: #3d708f;
    font-size: 0.86rem;
    font-weight: 700;
  }

  .message-card {
    margin: 0;
    border-radius: 14px;
    border: 1px solid #97dceb;
    background: rgba(255, 255, 255, 0.92);
    color: #35586e;
    font-size: 0.92rem;
    padding: 13px 14px;
    box-shadow: 0 10px 24px rgba(52, 113, 141, 0.12);
  }

  .message-card.is-error {
    border-color: #f4a7b1;
    color: #8e2f40;
    background: rgba(255, 245, 246, 0.95);
  }

  .message-card.debug {
    margin-top: 8px;
    font-size: 0.8rem;
    white-space: pre-wrap;
    word-break: break-word;
  }

  .group-sections {
    display: grid;
    gap: 16px;
  }

  .group-card {
    border-radius: 16px;
    border: 1px solid #9fdde8;
    background: rgba(255, 255, 255, 0.9);
    padding: 14px 14px 16px;
    box-shadow:
      0 12px 26px rgba(45, 112, 148, 0.12),
      inset 0 0 0 1px rgba(255, 255, 255, 0.8);
  }

  .card-cap {
    height: 6px;
    border-radius: 999px;
    margin-bottom: 10px;
    background: linear-gradient(90deg, #86d2ff 0%, #7fe9c4 100%);
  }

  h2 {
    margin: 0;
    color: #1f6288;
    font-size: 1.25rem;
  }

  .group-id {
    margin: 6px 0 0;
    color: #47708a;
    font-size: 0.78rem;
    font-weight: 700;
  }

  .group-description {
    margin: 8px 0 0;
    color: #3a5465;
    font-size: 0.9rem;
    line-height: 1.45;
    white-space: pre-wrap;
  }

  .channel {
    margin: 8px 0 0;
    color: #345b74;
    font-size: 0.85rem;
    display: flex;
    gap: 8px;
    flex-wrap: wrap;
    align-items: center;
  }

  .channel a {
    color: #246b8f;
    font-weight: 700;
  }

  .channel-status {
    border-radius: 999px;
    padding: 2px 8px;
    font-size: 0.74rem;
    font-weight: 700;
    background: rgba(160, 198, 219, 0.2);
    color: #4d6174;
  }

  .channel-status[data-connected='true'] {
    background: rgba(121, 227, 190, 0.25);
    color: #166a56;
  }

  .empty-text {
    margin: 12px 0 0;
    color: #4d6b7f;
    font-size: 0.88rem;
  }

  .video-list {
    list-style: none;
    margin: 12px 0 0;
    padding: 0;
    display: grid;
    gap: 12px;
  }

  .video-item {
    border-top: 1px solid rgba(134, 210, 255, 0.35);
    padding-top: 12px;
    display: grid;
    gap: 10px;
  }

  .video-media {
    border-radius: 12px;
    overflow: hidden;
    background: #eaf4fa;
  }

  .video-media iframe,
  .video-media img,
  .media-placeholder {
    width: 100%;
    aspect-ratio: 16 / 9;
    border: 0;
    display: block;
  }

  .media-placeholder {
    display: grid;
    place-items: center;
    color: #6a8397;
    font-size: 0.84rem;
  }

  .video-info h3 {
    margin: 0;
    color: #1f5e83;
    font-size: 1rem;
  }

  .video-info p {
    margin: 6px 0 0;
    color: #3a5465;
    font-size: 0.86rem;
    line-height: 1.45;
    white-space: pre-wrap;
  }

  .video-info a {
    display: inline-block;
    margin-top: 8px;
    color: #206f95;
    font-weight: 700;
    font-size: 0.85rem;
  }

  @media (min-width: 720px) {
    .page-bg {
      padding-top: 42px;
    }

    .hero-panel {
      padding: 30px 24px;
    }

    .video-item {
      grid-template-columns: minmax(240px, 330px) 1fr;
      align-items: start;
    }
  }
</style>

