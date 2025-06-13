<script lang="ts">
import { onMount } from 'svelte'
import Particles, { particlesInit } from '@tsparticles/svelte'
import { loadStarsPreset } from '@tsparticles/preset-stars'

let username = $state('')
let repo = $state('')
let stars = $state<number | null>(null)
let description = $state<string | null>(null)
let error = $state<string | null>(null)
let isLoading = $state(false)
let userRepos = $state<Array<{ name: string, stargazers_count: number, description: string | null }> | null>(null)

const STAR_TIERS = [
  { color: 'gold', value: 10000 },
  { color: 'green', value: 1000 },
  { color: 'blue', value: 100 },
  { color: 'blueviolet', value: 50 },
  { color: 'red', value: 10 },
  { color: 'silver', value: 1 }
]

const calculateStars = (count: number | null) => {
  if (count === null || count < 0) return []
  if (count < 10) return Array(count).fill('silver')

  let remainingStars = count
  const displayStars: string[] = []

  for (const tier of STAR_TIERS) {
    if (remainingStars >= tier.value) {
      const numTierStars = Math.floor(remainingStars / tier.value)
      for (let i = 0; i < numTierStars; i++) displayStars.push(tier.color)
      remainingStars %= tier.value
    }
  }

  return displayStars
}

let displayedStars = $derived(calculateStars(stars))

const updateRepoFromPath = () => {
  const path = window.location.pathname.slice(1)
  const parts = path.split('/')
  if (parts.length === 2 && parts[0] && parts[1]) {
    username = parts[0]
    repo = parts[1]
    error = null
    userRepos = null
  } else if (parts.length === 1 && parts[0]) {
    username = parts[0]
    repo = ''
    error = null
    userRepos = null
  } else if (path !== '') {
    username = ''
    repo = ''
    stars = null
    userRepos = null
    error = 'Invalid URL format. Use <strong>/username</strong> or <strong>/username/repo</strong>'
  } else {
    username = ''
    repo = ''
    stars = null
    userRepos = null
    error = 'Enter a path in the URL like <strong>/username</strong> or <strong>/username/repo</strong>'
  }
}

onMount(() => {
  updateRepoFromPath()
  window.addEventListener('popstate', updateRepoFromPath)
  return () => window.removeEventListener('popstate', updateRepoFromPath)
})

particlesInit(async (engine) => {
  await loadStarsPreset(engine)
})

let particleCount = $derived(() => {
  if (stars === null) return 50
  if (stars === 0) return 20
  if (stars < 10) return Math.min(stars * 3, 30)
  if (stars < 100) return Math.min(stars / 2, 80)
  if (stars < 1000) return Math.min(stars / 10, 150)
  return Math.min(stars / 50, 300)
})

let particlesOptions = $derived({
  preset: 'stars',
  background: {
    opacity: 0.8
  },
  detectRetina: true,
  fpsLimit: 60,
  particles: {
    number: {
      value: particleCount()
    },
    color: {
      value: ['#FFD700', '#87CEEB', '#98FB98', '#DDA0DD', '#F0E68C', '#C0C0C0']
    },
    opacity: {
      value: { min: 0.4, max: 1 },
      animation: {
        enable: true,
        speed: 3,
        sync: false
      }
    },
    size: {
      value: { min: 2, max: 4 },
      animation: {
        enable: true,
        speed: 4,
        sync: false
      }
    },
    stroke: {
      width: 1,
      color: {
        value: ['#FFFFFF', '#FFD700', '#87CEEB', '#98FB98', '#DDA0DD', '#F0E68C']
      },
      opacity: 0.8
    },
    shadow: {
      enable: true,
      color: '#FFFFFF',
      blur: 15,
      offset: {
        x: 0,
        y: 0
      }
    },
    twinkle: {
      particles: {
        enable: true,
        frequency: 0.05,
        opacity: 1
      },
      lines: {
        enable: true,
        frequency: 0.05,
        opacity: 1
      }
    },
    move: {
      enable: false
    }
  }
})

$effect(() => {
  if (username && repo) {
    isLoading = true
    stars = null
    description = null
    userRepos = null
    error = null
    fetch(`https://api.github.com/repos/${username}/${repo}`)
      .then(async (res) => {
        if (!res.ok) {
          const body = await res.json().catch(() => ({ message: 'Unknown error' }))
          throw new Error(body.message || `HTTP error! status: ${res.status}`)
        }

        return res.json()
      })
      .then((data) => {
        stars = data.stargazers_count
        description = data.description
      })
      .catch((err) => {
        console.error('Fetch error:', err)
        error = `Failed to fetch stars: ${err.message}`
        stars = null
        description = null
      })
      .finally(() => {
        isLoading = false
      })
  } else if (username && !repo) {
    isLoading = true
    stars = null
    description = null
    userRepos = null
    error = null
    fetch(`https://api.github.com/users/${username}/repos?per_page=100`)
      .then(async (res) => {
        if (!res.ok) {
          const body = await res.json().catch(() => ({ message: 'Unknown error' }))
          throw new Error(body.message || `HTTP error! status: ${res.status}`)
        }
        return res.json()
      })
      .then((data) => {
        if (!Array.isArray(data)) throw new Error('Unexpected response')
        userRepos = data
          .filter((repo) => repo.stargazers_count > 0)
          .map((repo) => ({
            name: repo.name,
            stargazers_count: repo.stargazers_count,
            description: repo.description
          }))
          .sort((a, b) => b.stargazers_count - a.stargazers_count)
      })
      .catch((err) => {
        console.error('Fetch error:', err)
        error = `Failed to fetch user repos: ${err.message}`
        userRepos = null
      })
      .finally(() => {
        isLoading = false
      })
  } else if (!error && !username && !repo) {
    stars = null
    description = null
    userRepos = null
    isLoading = false
  }
})
</script>

<Particles id="tsparticles" options={particlesOptions} />

<main>
  <h1>
    <a href="https://github.com/mathiscode/github-starcharts">GitHub Starcharts</a>
  </h1>

  <p class="made-with-love">Made with ❤️ by <a href="https://github.com/mathiscode">Jay Mathis</a></p>

  {#if username && repo}
    <div class="repo-card" style="margin: 2rem auto; max-width: 80vw;">
      <div class="repo-header">
        <h2>
          <a href={`/${username}`}>{username}</a> / <a
            href={`https://github.com/${username}/${repo}`}
            target="_blank"
            rel="noopener noreferrer"
          >{repo}</a>
        </h2>
      </div>

      {#if stars !== null}
        <div class="repo-star-count">{stars.toLocaleString()} stars</div>
      {/if}

      {#if description}
        <div class="repo-description">{description}</div>
      {/if}
    </div>
  {/if}

  {#if isLoading}
    <p>Loading stars...</p>
  {:else if error}
    <p class="error">{@html error}</p>
  {:else if stars !== null}
    <div class="stars-display">
      {#if displayedStars.length > 0}
        {#each displayedStars as starColor}
          <span title={STAR_TIERS.find(tier => tier.color === starColor)?.value.toLocaleString()} class="star {starColor}">★</span>
        {/each}
      {:else if stars === 0}
        <span>No stars yet!</span>
      {/if}
    </div>

    <div class="legend">
      <h3>Legend:</h3>
      <ul>
        {#each STAR_TIERS as tier}
          {#if tier.value >= 10}
            <li><span class="star {tier.color}">★</span> = {tier.value.toLocaleString()}</li>
          {/if}
        {/each}
        <li><span class="star silver">★</span> = 1</li>
      </ul>
    </div>
  {/if}

  {#if username && !repo && !isLoading && !error && userRepos}
    <h2>
      <a
        href={`https://github.com/${username}`}
        target="_blank"
        rel="noopener noreferrer"
      >{username}</a>'s repositories
    </h2>

    {#if userRepos.length === 0}
      <p>This user has no public repositories with stars.</p>
    {:else}
      <div class="user-repos-list">
        {#each userRepos as repo}
          <div class="repo-card">
            <div class="repo-header">
              <a
                href={`https://github.com/${username}/${repo.name}`}
                target="_blank"
                rel="noopener noreferrer"
                class="repo-name"
              >{repo.name}</a>
              <span class="repo-star-count">{repo.stargazers_count.toLocaleString()}</span>
            </div>
            <span class="repo-stars">
              {#each calculateStars(repo.stargazers_count) as starColor}
                <span class="star {starColor}">★</span>
              {/each}
            </span>
            {#if repo.description}
              <div class="repo-description">{repo.description}</div>
            {/if}
          </div>
        {/each}
      </div>
    {/if}
  {/if}

</main>

<style>
  :global(body) {
    background-color: #121212;
    color: #b0b0b0;
    font-family: system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
    margin: 0;
    padding: 2em;
  }

  :global(#tsparticles) {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: -1;
    pointer-events: none;
  }

  main {
    padding: 0;
    max-width: 900px;
    margin: 2em auto;
    text-align: center;
  }

  h1 {
    font-size: 3.5em;
    color: #ffffff;
    text-shadow: 0 0 10px rgba(255, 255, 255, 0.5),
                 0 0 20px rgba(255, 255, 255, 0.3),
                 0 0 30px rgba(255, 255, 255, 0.2);
    letter-spacing: 2px;
    font-weight: 700;
    margin-bottom: 0;
    background: linear-gradient(45deg, #ffffff, #a0a0a0);
    -webkit-background-clip: text;
    background-clip: text;
    -webkit-text-fill-color: transparent;
    animation: glow 3s ease-in-out infinite alternate;
  }

  @keyframes glow {
    from {
      text-shadow: 0 0 10px rgba(255, 255, 255, 0.5),
                   0 0 20px rgba(255, 255, 255, 0.3),
                   0 0 30px rgba(255, 255, 255, 0.2);
    }
    to {
      text-shadow: 0 0 20px rgba(255, 255, 255, 0.7),
                   0 0 30px rgba(255, 255, 255, 0.5),
                   0 0 40px rgba(255, 255, 255, 0.3);
    }
  }

  h2 {
    font-size: 1.2em;
    color: #777777;
    margin-top: 0;
    margin-bottom: 0;
  }

  h3 {
    color: #999999;
  }

  .description {
    font-style: italic;
    font-size: 0.9em;
    max-width: 60ch;
    margin: 0 auto 2em auto;
    color: #888888;
    line-height: 1.4;
  }

  .legend {
    background-color: #1f1f1f;
    padding: 0.8em 1em;
    border-radius: 6px;
    margin: 2em auto;
    display: inline-block;
    text-align: left;
    border: 1px solid #333;
    font-size: 0.85em;
  }

  .legend h3 {
    font-size: 1em;
    margin-top: 0;
    margin-bottom: 0.6em;
    color: #aaaaaa;
  }

  .legend ul {
    list-style: none;
    padding: 0;
    margin: 0;
  }

  .legend li {
    display: flex;
    align-items: center;
    margin-bottom: 0.4em;
    color: #a0a0a0;
  }

  .legend .star {
    font-size: 1.2em;
  }

  @media (min-width: 768px) {
    .legend {
      max-width: 100%;
      padding: 0.8em 1.5em;
    }

    .legend h3 {
      display: none;
    }

    .legend ul {
      display: flex;
      flex-wrap: wrap;
      justify-content: center;
      gap: 1.5em;
    }

    .legend li {
      margin-bottom: 0;
    }
  }

  .stars-display {
    margin: 2em 0;
    padding: 2em;
    background-color: #252525;
    border-radius: 12px;
    border: 1px solid #444;
    box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2);
  }

  .star {
    margin: 0 0.1em;
    line-height: 1;
    display: inline-block;
    vertical-align: middle;
    text-shadow: 0 2px 4px rgba(0, 0, 0, 0.3);
    cursor: help;
  }

  .stars-display .star.gold    { color: gold;        font-size: 4.8em; }
  .stars-display .star.green   { color: lightgreen;  font-size: 4.2em; }
  .stars-display .star.blue    { color: lightskyblue; font-size: 3.8em; }
  .stars-display .star.blueviolet  { color: blueviolet;     font-size: 3.5em; }
  .stars-display .star.red     { color: tomato;      font-size: 3.0em; }
  .stars-display .star.silver  { color: silver;      font-size: 2.5em; }

  .legend .star.gold    { color: gold; }
  .legend .star.green   { color: lightgreen; }
  .legend .star.blue    { color: lightskyblue; }
  .legend .star.blueviolet  { color: blueviolet; }
  .legend .star.red     { color: tomato; }
  .legend .star.silver  { color: silver; }

  .error {
    color: tomato;
    background-color: #442222;
    padding: 1em;
    border-radius: 4px;
    display: inline-block;
    margin-top: 2em;
  }

  .user-repos-list {
    display: flex;
    flex-wrap: wrap;
    gap: 2em;
    justify-content: center;
    margin: 2em 0;
    align-items: stretch;
  }

  .repo-card {
    background: #232323;
    border: 1px solid #333;
    border-radius: 10px;
    padding: 1.2em 1.5em;
    width: 320px;
    min-width: 320px;
    max-width: 320px;
    box-shadow: 0 2px 8px rgba(0,0,0,0.15);
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: flex-start;
  }

  .repo-header {
    display: flex;
    align-items: center;
    gap: 0.7em;
    margin-bottom: 0.5em;
  }

  .repo-star-count {
    color: #ffd700;
    font-size: 1.1em;
    font-weight: bold;
    letter-spacing: 0.5px;
  }

  .repo-name {
    font-size: 1.1em;
    color: #b0b0ff;
    font-weight: bold;
    text-decoration: none;
  }

  .repo-name:hover {
    text-decoration: underline;
  }

  .repo-stars {
    margin-bottom: 0.5em;
    color: #aaa;
    font-size: 1em;
  }

  .repo-card .star.gold    { color: gold; }
  .repo-card .star.green   { color: lightgreen; }
  .repo-card .star.blue    { color: lightskyblue; }
  .repo-card .star.blueviolet  { color: blueviolet; }
  .repo-card .star.red     { color: tomato; }
  .repo-card .star.silver  { color: silver; }
  .repo-card .star {
    font-size: 1.5em;
  }

  .repo-description {
    color: #888;
    font-size: 0.95em;
    text-align: center;
    margin-top: 0.5em;
  }

  .made-with-love {
    font-size: 0.8em;
    color: #888888;
    margin-top: 0;
  }
</style>
