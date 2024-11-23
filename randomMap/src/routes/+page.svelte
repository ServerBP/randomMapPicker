<script lang="ts">
    import { onMount } from 'svelte';

    interface MapDetails {
        id: string;
        name: string;
        description: string | null;
        curator: string | null;
        uploaded: string;
        automapper: boolean;
        ranked: boolean;
        qualified: boolean;
        versions: Array<{
            coverURL: string;
            diffs: Array<{
                difficulty: string;
                stars: number;
            }>;
        }>;
        metadata: {
            bpm: number;
            duration: number;
            songName: string;
            songSubName: string;
            songAuthorName: string;
            levelAuthorName: string;
        };
        stats: {
            plays: number;
            downloads: number;
            upvotes: number;
            downvotes: number;
        };
    }

    let randomMap: MapDetails | null = null;
    let isLoading = false;
    let error: string | null = null;

    async function fetchLatestMapId(): Promise<string> {
        const response = await fetch('https://api.beatsaver.com/maps/latest');
        if (!response.ok) throw new Error('Failed to fetch latest map');
        const data = await response.json();
        return data.docs[0].id;
    }

    async function checkMapExists(mapId: string): Promise<boolean> {
        try {
            const response = await fetch(`https://api.beatsaver.com/maps/id/${mapId}`);
            return response.ok;
        } catch {
            return false;
        }
    }

    function hexToDecimal(hex: string): number {
        return parseInt(hex, 16);
    }

    function decimalToHex(decimal: number): string {
        return decimal.toString(16).padStart(1, '0');
    }

    function getRandomNumber(min: number, max: number): number {
        return Math.floor(Math.random() * (max - min + 1)) + min;
    }

    function formatDuration(seconds: number): string {
        const minutes = Math.floor(seconds / 60);
        const remainingSeconds = Math.floor(seconds % 60);
        return `${minutes}:${remainingSeconds.toString().padStart(2, '0')}`;
    }

    function formatDate(dateString: string): string {
        return new Date(dateString).toLocaleDateString(undefined, {
            year: 'numeric',
            month: 'short',
            day: 'numeric'
        });
    }

    async function findRandomMap() {
        isLoading = true;
        error = null;
        randomMap = null;

        try {
            const latestMapId = await fetchLatestMapId();
            const maxDecimal = hexToDecimal(latestMapId);
            
            let attempts = 0;
            const maxAttempts = 10;
            
            while (attempts < maxAttempts) {
                const randomDecimal = getRandomNumber(1, maxDecimal);
                const randomHex = decimalToHex(randomDecimal);
                
                const exists = await checkMapExists(randomHex);
                if (exists) {
                    const response = await fetch(`https://api.beatsaver.com/maps/id/${randomHex}`);
                    randomMap = await response.json();
                    break;
                }
                
                attempts++;
            }
            
            if (!randomMap) {
                throw new Error('Could not find a valid map after multiple attempts');
            }
        } catch (e) {
            error = e instanceof Error ? e.message : 'An unexpected error occurred';
        } finally {
            isLoading = false;
        }
    }

    onMount(() => {
        // You can auto-load a random map on mount if desired
        // findRandomMap();
    });
</script>

<div class="container">
    <div class="content">
        <h1>Beat Saber Random Map Picker</h1>
        
        <div class="card">
            <button
                on:click={findRandomMap}
                class="primary-button"
                class:loading={isLoading}
                disabled={isLoading}
            >
                {#if isLoading}
                    <span class="spinner">↻</span>
                    Finding random map...
                {:else}
                    Get Random Map
                {/if}
            </button>

            {#if error}
                <div class="error-message">
                    <p>Error: {error}</p>
                </div>
            {/if}

            {#if randomMap}
                <div class="map-details">
                    <div class="map-header">
                        <img
                            src={randomMap.versions[0].coverURL}
                            alt="Map cover"
                            class="cover-image"
                        />
                        <div class="map-title">
                            <h2>{randomMap.name}</h2>
                            <p class="subtitle">
                                {randomMap.metadata.songAuthorName} - {randomMap.metadata.songName}
                                {#if randomMap.metadata.songSubName}
                                    ({randomMap.metadata.songSubName})
                                {/if}
                            </p>
                        </div>
                    </div>

                    <div class="map-info-grid">
                        <div class="info-item">
                            <span class="label">Mapper</span>
                            <span>{randomMap.metadata.levelAuthorName}</span>
                        </div>
                        <div class="info-item">
                            <span class="label">BPM</span>
                            <span>{randomMap.metadata.bpm}</span>
                        </div>
                        <div class="info-item">
                            <span class="label">Duration</span>
                            <span>{formatDuration(randomMap.metadata.duration)}</span>
                        </div>
                        <div class="info-item">
                            <span class="label">Uploaded</span>
                            <span>{formatDate(randomMap.uploaded)}</span>
                        </div>
                    </div>

                    <div class="stats-container">
                        <div class="stat">
                            <span class="stat-value">{randomMap.stats.downloads.toLocaleString()}</span>
                            <span class="stat-label">Downloads</span>
                        </div>
                        <div class="stat">
                            <span class="stat-value">{randomMap.stats.plays.toLocaleString()}</span>
                            <span class="stat-label">Plays</span>
                        </div>
                        <div class="stat">
                            <span class="stat-value">{randomMap.stats.upvotes}</span>
                            <span class="stat-label">Upvotes</span>
                        </div>
                    </div>

                    <div class="difficulties">
                        <span class="label">Difficulties:</span>
                        <div class="difficulty-tags">
                            {#each randomMap.versions[0].diffs as diff}
                                <span class="difficulty-tag" data-difficulty={diff.difficulty.toLowerCase()}>
                                    {diff.difficulty} ({diff.stars}★)
                                </span>
                            {/each}
                        </div>
                    </div>

                    {#if randomMap.description}
                        <div class="description">
                            <span class="label">Description:</span>
                            <p>{randomMap.description}</p>
                        </div>
                    {/if}

                    <div class="map-badges">
                        {#if randomMap.ranked}
                            <span class="badge ranked">Ranked</span>
                        {/if}
                        {#if randomMap.qualified}
                            <span class="badge qualified">Qualified</span>
                        {/if}
                        {#if randomMap.automapper}
                            <span class="badge automapper">AI Generated</span>
                        {/if}
                    </div>

                    <div class="button-container">
                        <a
                            href="https://beatsaver.com/maps/{randomMap.id}"
                            target="_blank"
                            rel="noopener noreferrer"
                            class="secondary-button"
                        >
                            View on BeatSaver
                        </a>
                    </div>
                </div>
            {/if}
        </div>
    </div>
</div>

<style>
    :global(body) {
        margin: 0;
        font-family: system-ui, -apple-system, sans-serif;
        background-color: #1a1a1a;
        color: #ffffff;
    }

    .container {
        min-height: 100vh;
        padding: 2rem;
    }

    .content {
        max-width: 800px;
        margin: 0 auto;
    }

    h1 {
        font-size: 2.5rem;
        font-weight: bold;
        text-align: center;
        margin-bottom: 2rem;
        color: #ffffff;
    }

    .card {
        background-color: #2a2a2a;
        border-radius: 1rem;
        padding: 1.5rem;
        box-shadow: 0 4px 6px rgba(0, 0, 0, 0.3);
    }

    .primary-button {
        width: 100%;
        background-color: #3b82f6;
        color: white;
        font-weight: 500;
        padding: 0.75rem 1.5rem;
        border-radius: 0.5rem;
        border: none;
        cursor: pointer;
        transition: background-color 0.2s;
        margin-bottom: 1.5rem;
    }

    .primary-button:hover {
        background-color: #2563eb;
    }

    .primary-button:disabled {
        opacity: 0.5;
        cursor: not-allowed;
    }

    .spinner {
        display: inline-block;
        animation: spin 1s linear infinite;
        margin-right: 0.5rem;
    }

    @keyframes spin {
        from { transform: rotate(0deg); }
        to { transform: rotate(360deg); }
    }

    .error-message {
        background-color: rgba(220, 38, 38, 0.2);
        color: #fecaca;
        padding: 1rem;
        border-radius: 0.5rem;
        margin-bottom: 1rem;
    }

    .map-header {
        display: flex;
        gap: 1.5rem;
        margin-bottom: 1.5rem;
    }

    .cover-image {
        width: 150px;
        height: 150px;
        border-radius: 0.5rem;
        object-fit: cover;
    }

    .map-title {
        flex: 1;
    }

    .map-title h2 {
        font-size: 1.5rem;
        font-weight: 600;
        margin: 0 0 0.5rem 0;
    }

    .subtitle {
        color: #9ca3af;
        margin: 0;
    }

    .map-info-grid {
        display: grid;
        grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
        gap: 1rem;
        margin-bottom: 1.5rem;
    }

    .info-item {
        background-color: #333333;
        padding: 0.75rem;
        border-radius: 0.5rem;
    }

    .label {
        display: block;
        color: #9ca3af;
        font-size: 0.875rem;
        margin-bottom: 0.25rem;
    }

    .stats-container {
        display: flex;
        justify-content: space-around;
        margin-bottom: 1.5rem;
        background-color: #333333;
        padding: 1rem;
        border-radius: 0.5rem;
    }

    .stat {
        text-align: center;
    }

    .stat-value {
        display: block;
        font-size: 1.25rem;
        font-weight: 600;
        color: #3b82f6;
    }

    .stat-label {
        font-size: 0.875rem;
        color: #9ca3af;
    }

    .difficulties {
        margin-bottom: 1.5rem;
    }

    .difficulty-tags {
        display: flex;
        flex-wrap: wrap;
        gap: 0.5rem;
        margin-top: 0.5rem;
    }

    .difficulty-tag {
        padding: 0.25rem 0.75rem;
        border-radius: 9999px;
        font-size: 0.875rem;
        font-weight: 500;
    }

    .difficulty-tag[data-difficulty="easy"] { background-color: #059669; }
    .difficulty-tag[data-difficulty="normal"] { background-color: #2563eb; }
    .difficulty-tag[data-difficulty="hard"] { background-color: #dc2626; }
    .difficulty-tag[data-difficulty="expert"] { background-color: #7c3aed; }
    .difficulty-tag[data-difficulty="expertplus"] { background-color: #db2777; }

    .description {
        background-color: #333333;
        padding: 1rem;
        border-radius: 0.5rem;
        margin-bottom: 1.5rem;
    }

    .description p {
        margin: 0.5rem 0 0 0;
        white-space: pre-wrap;
    }

    .map-badges {
        display: flex;
        gap: 0.5rem;
        margin-bottom: 1.5rem;
    }

    .badge {
        padding: 0.25rem 0.75rem;
        border-radius: 9999px;
        font-size: 0.875rem;
        font-weight: 500;
    }

    .badge.ranked { background-color: #059669; }
    .badge.qualified { background-color: #2563eb; }
    .badge.automapper { background-color: #7c3aed; }

    .button-container {
        display: flex;
        gap: 1rem;
    }

    .secondary-button {
        flex: 1;
        display: inline-block;
        background-color: #059669;
        color: white;
        text-decoration: none;
        font-weight: 500;
        padding: 0.75rem 1rem;
        border-radius: 0.5rem;
        text-align: center;
        transition: background-color 0.2s;
    }

    .secondary-button:hover {
        background-color: #047857;
    }

    .playlist-button {
        background-color: #7c3aed;
    }

    .playlist-button:hover {
        background-color: #6d28d9;
    }

    @media (max-width: 640px) {
        .container {
            padding: 1rem;
        }

        h1 {
            font-size: 2rem;
        }

        .map-header {
            flex-direction: column;
            align-items: center;
            text-align: center;
        }

        .button-container {
            flex-direction: column;
        }
    }
</style>