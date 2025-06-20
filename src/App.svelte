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
let avatarUrl = $state<string | null>(null)

let formUsername = $state('')
let formRepo = $state('')
let suggestions = $state<Array<{ login: string, avatar_url: string, type: string }>>([])
let showSuggestions = $state(false)
let isSearching = $state(false)
let searchTimeout: number | null = null

let repoSuggestions = $state<Array<{ name: string, full_name: string, description: string | null, stargazers_count: number }>>([])
let showRepoSuggestions = $state(false)
let isSearchingRepos = $state(false)
let repoSearchTimeout: number | null = null

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
  const hash = window.location.hash.slice(1)
  const parts = hash.split('/')
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
  } else if (hash !== '') {
    username = ''
    repo = ''
    stars = null
    userRepos = null
    error = 'Invalid URL format. Use <strong>#username</strong> or <strong>#username/repo</strong>'
  } else {
    username = ''
    repo = ''
    stars = null
    userRepos = null
    error = 'Enter a hash in the URL like <strong>#username</strong> or <strong>#username/repo</strong>'
  }
}

onMount(() => {
  updateRepoFromPath()
  window.addEventListener('hashchange', updateRepoFromPath)
  return () => window.removeEventListener('hashchange', updateRepoFromPath)
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

const handleFormSubmit = (e: Event) => {
  e.preventDefault()
  if (!formUsername.trim()) return
  
  const newHash = formRepo.trim() 
    ? `${formUsername.trim()}/${formRepo.trim()}`
    : formUsername.trim()
  
  window.location.hash = newHash
  showSuggestions = false
}

const searchUsers = async (query: string) => {
  if (query.length < 3) {
    suggestions = []
    showSuggestions = false
    return
  }

  try {
    isSearching = true
    const response = await fetch(`https://api.github.com/search/users?q=${encodeURIComponent(query)}&per_page=8`)
    
    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`)
    }
    
    const data = await response.json()
    suggestions = data.items.map((user: any) => ({
      login: user.login,
      avatar_url: user.avatar_url,
      type: user.type
    }))
    showSuggestions = suggestions.length > 0
  } catch (error) {
    console.error('Error searching users:', error)
    suggestions = []
    showSuggestions = false
  } finally {
    isSearching = false
  }
}

const handleUsernameInput = (e: Event) => {
  const target = e.target as HTMLInputElement
  formUsername = target.value
  
  if (searchTimeout) {
    clearTimeout(searchTimeout)
  }
  
  searchTimeout = setTimeout(() => {
    searchUsers(formUsername)
  }, 300)
}

const selectSuggestion = (login: string) => {
  formUsername = login
  showSuggestions = false
  suggestions = []
  
  if (searchTimeout) {
    clearTimeout(searchTimeout)
  }
}

const searchRepositories = async (query: string, username: string) => {
  if (query.length < 2) {
    repoSuggestions = []
    showRepoSuggestions = false
    return
  }

  try {
    isSearchingRepos = true
    
    const searchQuery = username.trim() 
      ? `user:${username.trim()} ${query} in:name`
      : `${query} in:name`
    
    const response = await fetch(`https://api.github.com/search/repositories?q=${encodeURIComponent(searchQuery)}&per_page=8&sort=stars&order=desc`)
    
    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`)
    }
    
    const data = await response.json()
    repoSuggestions = data.items.map((repo: any) => ({
      name: repo.name,
      full_name: repo.full_name,
      description: repo.description,
      stargazers_count: repo.stargazers_count
    }))
    showRepoSuggestions = repoSuggestions.length > 0
  } catch (error) {
    console.error('Error searching repositories:', error)
    repoSuggestions = []
    showRepoSuggestions = false
  } finally {
    isSearchingRepos = false
  }
}

const handleRepoInput = (e: Event) => {
  const target = e.target as HTMLInputElement
  formRepo = target.value
  
  if (repoSearchTimeout) {
    clearTimeout(repoSearchTimeout)
  }
  
  repoSearchTimeout = setTimeout(() => {
    searchRepositories(formRepo, formUsername)
  }, 300)
}

const selectRepoSuggestion = (repoName: string) => {
  formRepo = repoName
  showRepoSuggestions = false
  repoSuggestions = []
  
  if (repoSearchTimeout) {
    clearTimeout(repoSearchTimeout)
  }
}

const fetchUserRepositories = async (username: string) => {
  if (!username.trim()) {
    repoSuggestions = []
    showRepoSuggestions = false
    return
  }

  try {
    isSearchingRepos = true
    
    const response = await fetch(`https://api.github.com/users/${username.trim()}/repos?per_page=10&sort=updated&direction=desc`)
    
    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`)
    }
    
    const data = await response.json()
    
    // Sort by star count descending and take top 10
    const sortedRepos = data
      .sort((a: any, b: any) => b.stargazers_count - a.stargazers_count)
      .slice(0, 10)
    
    repoSuggestions = sortedRepos.map((repo: any) => ({
      name: repo.name,
      full_name: repo.full_name,
      description: repo.description,
      stargazers_count: repo.stargazers_count
    }))
    
    showRepoSuggestions = repoSuggestions.length > 0
  } catch (error) {
    console.error('Error fetching user repositories:', error)
    repoSuggestions = []
    showRepoSuggestions = false
  } finally {
    isSearchingRepos = false
  }
}

const handleRepoFocus = () => {
  if (repoSuggestions.length > 0) {
    showRepoSuggestions = true
  } else if (formUsername.trim() && !formRepo.trim()) {
    // If username exists but no repo typed yet, fetch default suggestions
    fetchUserRepositories(formUsername)
  }
}

$effect(() => {
  if (username && repo) {
    isLoading = true
    stars = null
    description = null
    userRepos = null
    avatarUrl = null
    error = null
    
    Promise.all([
      fetch(`https://api.github.com/repos/${username}/${repo}`),
      fetch(`https://api.github.com/users/${username}`)
    ])
      .then(async ([repoRes, userRes]) => {
        if (!repoRes.ok) {
          const body = await repoRes.json().catch(() => ({ message: 'Unknown error' }))
          throw new Error(body.message || `HTTP error! status: ${repoRes.status}`)
        }
        if (!userRes.ok) {
          const body = await userRes.json().catch(() => ({ message: 'Unknown error' }))
          throw new Error(body.message || `HTTP error! status: ${userRes.status}`)
        }

        const [repoData, userData] = await Promise.all([
          repoRes.json(),
          userRes.json()
        ])
        
        return { repoData, userData }
      })
      .then(({ repoData, userData }) => {
        stars = repoData.stargazers_count
        description = repoData.description
        avatarUrl = userData.avatar_url
      })
      .catch((err) => {
        console.error('Fetch error:', err)
        error = `Failed to fetch data: ${err.message}`
        stars = null
        description = null
        avatarUrl = null
      })
      .finally(() => {
        isLoading = false
      })
  } else if (username && !repo) {
    isLoading = true
    stars = null
    description = null
    userRepos = null
    avatarUrl = null
    error = null
    
    Promise.all([
      fetch(`https://api.github.com/users/${username}/repos?per_page=100`),
      fetch(`https://api.github.com/users/${username}`)
    ])
      .then(async ([reposRes, userRes]) => {
        if (!reposRes.ok) {
          const body = await reposRes.json().catch(() => ({ message: 'Unknown error' }))
          throw new Error(body.message || `HTTP error! status: ${reposRes.status}`)
        }
        if (!userRes.ok) {
          const body = await userRes.json().catch(() => ({ message: 'Unknown error' }))
          throw new Error(body.message || `HTTP error! status: ${userRes.status}`)
        }
        
        const [reposData, userData] = await Promise.all([
          reposRes.json(),
          userRes.json()
        ])
        
        return { reposData, userData }
      })
      .then(({ reposData, userData }) => {
        if (!Array.isArray(reposData)) throw new Error('Unexpected response')
        userRepos = reposData
          .filter((repo) => repo.stargazers_count > 0)
          .map((repo) => ({
            name: repo.name,
            stargazers_count: repo.stargazers_count,
            description: repo.description
          }))
          .sort((a, b) => b.stargazers_count - a.stargazers_count)
        avatarUrl = userData.avatar_url
      })
      .catch((err) => {
        console.error('Fetch error:', err)
        error = `Failed to fetch user data: ${err.message}`
        userRepos = null
        avatarUrl = null
      })
      .finally(() => {
        isLoading = false
      })
  } else if (!error && !username && !repo) {
    stars = null
    description = null
    userRepos = null
    avatarUrl = null
    isLoading = false
  }
})
</script>

<Particles id="tsparticles" options={particlesOptions} />

<main>
  <h1>
    <!-- svelte-ignore a11y_invalid_attribute -->
    <a href="#">GitHub Starcharts</a>
  </h1>

  <p class="made-with-love">
    <a
      href="https://github.com/mathiscode/github-starcharts"
      target="_blank"
      rel="noopener noreferrer"
    >Made</a> with ❤️ by 
    <a
      href="https://github.com/mathiscode"
      target="_blank"
      rel="noopener noreferrer"
    >Jay Mathis</a>
  </p>

  <form class="navigation-form" onsubmit={handleFormSubmit}>
    <div class="form-row">
      <div class="username-input-container">
        <input
          type="text"
          bind:value={formUsername}
          oninput={handleUsernameInput}
          onfocus={() => { if (suggestions.length > 0) showSuggestions = true }}
          onblur={() => setTimeout(() => showSuggestions = false, 150)}
          placeholder="GitHub username"
          class="username-input"
          required
        />
        {#if showSuggestions && suggestions.length > 0}
          <div class="suggestions-dropdown">
            {#each suggestions as suggestion}
              <button
                type="button"
                class="suggestion-item"
                onclick={() => selectSuggestion(suggestion.login)}
              >
                <img src={suggestion.avatar_url} alt={suggestion.login} class="avatar" />
                <span class="login">{suggestion.login}</span>
                <span class="type">{suggestion.type}</span>
              </button>
            {/each}
          </div>
        {/if}
        {#if isSearching}
          <div class="search-indicator">Searching...</div>
        {/if}
      </div>
      <div class="repo-input-container">
        <input
          type="text"
          bind:value={formRepo}
          oninput={handleRepoInput}
          onfocus={handleRepoFocus}
          onblur={() => setTimeout(() => showRepoSuggestions = false, 150)}
          placeholder="Repository (optional)"
          class="repo-input"
        />
        {#if showRepoSuggestions && repoSuggestions.length > 0}
          <div class="repo-suggestions-dropdown">
            {#each repoSuggestions as suggestion}
              <button
                type="button"
                class="repo-suggestion-item"
                onclick={() => selectRepoSuggestion(suggestion.name)}
              >
                <div class="repo-suggestion-main">
                  <span class="repo-suggestion-name">{suggestion.name}</span>
                  <span class="repo-suggestion-stars">★ {suggestion.stargazers_count}</span>
                </div>
                {#if suggestion.description}
                  <div class="repo-suggestion-description">{suggestion.description}</div>
                {/if}
              </button>
            {/each}
          </div>
        {/if}
        {#if isSearchingRepos}
          <div class="repo-search-indicator">Searching repos...</div>
        {/if}
      </div>
      <button class="go-button" type="submit" disabled={!formUsername.trim()}>
        Go
      </button>
    </div>
  </form>

  {#if avatarUrl && (username && (repo || userRepos))}
    <div class="user-avatar-section">
      <img src={avatarUrl} alt={username} class="user-avatar-large" />
      <div class="username-display">{username}</div>
    </div>
  {/if}

  {#if username && repo}
    <div class="repo-card" style="margin: 2rem auto; max-width: 80vw;">
      <div class="repo-header">
        <h2>
          <a href={`#${username}`}>{username}</a> / <a
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

    <div class="stars-display">
      {#if displayedStars.length > 0}
        {#each displayedStars as starColor}
          <span title={STAR_TIERS.find(tier => tier.color === starColor)?.value.toLocaleString()} class="star {starColor}">★</span>
        {/each}
      {:else if stars === 0}
        <span>No stars yet!</span>
      {/if}
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
                href={`#${username}/${repo.name}`}
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
    /* background-color: #252525; */
    /* border-radius: 12px; */
    /* border: 1px solid #444; */
    /* box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2); */
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

  .user-avatar-section {
    display: flex;
    flex-direction: column;
    align-items: center;
    margin: 2rem 0;
    gap: 0.8rem;
  }

  .user-avatar-large {
    width: 80px;
    height: 80px;
    border-radius: 50%;
    border: 3px solid #444;
    box-shadow: 0 4px 15px rgba(0, 0, 0, 0.4);
    transition: transform 0.2s ease;
  }

  .user-avatar-large:hover {
    transform: scale(1.05);
  }

  .username-display {
    font-size: 1.3em;
    font-weight: 600;
    color: #b0b0ff;
    text-shadow: 0 2px 4px rgba(0, 0, 0, 0.3);
  }

  .navigation-form {
    margin: 2rem auto 3rem auto;
    max-width: 500px;
  }

  .form-row {
    display: flex;
    gap: 0.5rem;
    align-items: center;
    flex-wrap: wrap;
    justify-content: center;
  }

  .username-input-container, .repo-input-container {
    position: relative;
    flex: 0 0 auto;
    width: 200px;
  }

  .repo-input-container {
    width: 220px;
  }

  .username-input, .repo-input {
    padding: 0.8rem 1rem;
    border: 2px solid #333;
    border-radius: 6px;
    background-color: #1a1a1a;
    color: #ffffff;
    font-size: 1rem;
    transition: border-color 0.2s ease;
    width: 100%;
    box-sizing: border-box;
  }

  .username-input:focus, .repo-input:focus {
    outline: none;
    border-color: #5555ff;
    box-shadow: 0 0 0 2px rgba(85, 85, 255, 0.2);
  }

  .username-input::placeholder, .repo-input::placeholder {
    color: #666;
  }

  .navigation-form button {
    padding: 0.8rem 1.5rem;
    border: none;
    border-radius: 6px;
    background: linear-gradient(45deg, #5555ff, #7777ff);
    color: white;
    font-size: 1rem;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.2s ease;
    min-width: 60px;
  }

  .navigation-form button:hover:not(:disabled) {
    background: linear-gradient(45deg, #6666ff, #8888ff);
    transform: translateY(-1px);
    box-shadow: 0 4px 12px rgba(85, 85, 255, 0.3);
  }

  .navigation-form button:disabled {
    background: #333;
    color: #666;
    cursor: not-allowed;
    transform: none;
    box-shadow: none;
  }

  .suggestions-dropdown {
    position: absolute;
    top: 100%;
    left: 0;
    background: #1a1a1a;
    border: 2px solid #333;
    border-top: none;
    border-radius: 0 0 6px 6px;
    max-height: 300px;
    overflow-y: auto;
    z-index: 1000;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
    min-width: 300px;
    width: max-content;
  }

  .suggestion-item {
    width: 100%;
    padding: 0.8rem 1rem;
    margin-bottom: 0.3rem;
    border: none;
    background: transparent;
    color: #ffffff;
    text-align: left;
    cursor: pointer;
    display: flex;
    align-items: center;
    gap: 0.8rem;
    transition: background-color 0.2s ease;
    font-size: 0.95rem;
  }

  .suggestion-item:last-child {
    margin-bottom: 0;
  }

  .suggestion-item:hover {
    background-color: #2a2a2a;
  }

  .suggestion-item:focus {
    outline: none;
    background-color: #333;
  }

  .suggestion-item .avatar {
    width: 24px;
    height: 24px;
    border-radius: 50%;
    flex-shrink: 0;
  }

  .suggestion-item .login {
    font-weight: 600;
    color: #b0b0ff;
    flex: 1;
  }

  .suggestion-item .type {
    font-size: 0.8em;
    color: #888;
    text-transform: capitalize;
    padding: 0.2rem 0.5rem;
    background: #333;
    border-radius: 4px;
    flex-shrink: 0;
  }

  .search-indicator {
    position: absolute;
    top: 100%;
    left: 0;
    background: #1a1a1a;
    border: 2px solid #333;
    border-top: none;
    border-radius: 0 0 6px 6px;
    padding: 0.8rem 1rem;
    color: #888;
    font-size: 0.9rem;
    z-index: 999;
    min-width: 300px;
    width: max-content;
  }

  .repo-search-indicator {
    position: absolute;
    top: 100%;
    right: 0;
    background: #1a1a1a;
    border: 2px solid #333;
    border-top: none;
    border-radius: 0 0 6px 6px;
    padding: 0.8rem 1rem;
    color: #888;
    font-size: 0.9rem;
    z-index: 999;
    width: 350px;
    max-width: calc(100vw - 2rem);
  }

  .repo-suggestions-dropdown {
    position: absolute;
    top: 100%;
    right: 0;
    background: #1a1a1a;
    border: 2px solid #333;
    border-top: none;
    border-radius: 0 0 6px 6px;
    max-height: 300px;
    overflow-y: auto;
    z-index: 1000;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
    width: 350px;
    max-width: calc(100vw - 2rem);
  }

  .repo-suggestion-item {
    width: 100%;
    padding: 0.8rem 1rem;
    margin-bottom: 0.3rem;
    border: none;
    background: transparent;
    color: #ffffff;
    text-align: left;
    cursor: pointer;
    display: flex;
    flex-direction: column;
    gap: 0.3rem;
    transition: background-color 0.2s ease;
    font-size: 0.95rem;
  }

  .repo-suggestion-item:last-child {
    margin-bottom: 0;
  }

  .repo-suggestion-item:hover {
    background-color: #2a2a2a;
  }

  .repo-suggestion-item:focus {
    outline: none;
    background-color: #333;
  }

  .repo-suggestion-main {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 1rem;
  }

  .repo-suggestion-name {
    font-weight: 600;
    color: #ffffff;
    flex: 1;
  }

  .repo-suggestion-stars {
    font-size: 0.85em;
    color: #ffd700;
    font-weight: 500;
    flex-shrink: 0;
  }

  .repo-suggestion-description {
    font-size: 0.8em;
    color: #cccccc;
    line-height: 1.3;
    margin-top: 0.2rem;
  }

  @media (max-width: 600px) {
    .form-row {
      flex-direction: column;
      width: 100%;
    }

    .username-input-container, .repo-input-container {
      width: 100%;
      max-width: 300px;
    }

    .navigation-form button {
      width: 100%;
      max-width: 300px;
    }
  }

  .go-button {
    width: 100%;
    max-width: 425px;
  }
</style>
