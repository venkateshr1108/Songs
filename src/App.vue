<script setup>
import { computed, onMounted, onUnmounted, ref, watch } from 'vue'

const STORAGE_KEY = 'songbook-studio-vue-v1'
const songs = ref([])
const queue = ref([])
const activeTab = ref('library')
const librarySearch = ref('')
const builderSearch = ref('')
const songId = ref('')
const title = ref('')
const artist = ref('')
const lyrics = ref('')
const deletingId = ref('')
const showDeleteDialog = ref(false)
const toastMessage = ref('')
const fontSize = ref(12)
const headingSize = ref(18)
const headingFont = ref('Georgia, serif')
const paperSize = ref('letter')
const jsonInput = ref(null)
let toastTimer

const headingFonts = [
  { label: 'Classic serif', value: 'Georgia, serif' },
  { label: 'Clean sans serif', value: 'Arial, sans-serif' },
  { label: 'Rounded', value: "'Trebuchet MS', sans-serif" },
  { label: 'Typewriter', value: "'Courier New', monospace" }
]

const filteredSongs = computed(() => {
  const query = librarySearch.value.trim().toLocaleLowerCase()
  return songs.value.filter(song => matches(song, query))
})
const builderSongs = computed(() => {
  const query = builderSearch.value.trim().toLocaleLowerCase()
  return songs.value.filter(song => matches(song, query))
})
const queuedSongs = computed(() => queue.value.map(id => songs.value.find(song => song.id === id)).filter(Boolean))
const songCountText = computed(() => `${songs.value.length} song${songs.value.length === 1 ? '' : 's'}`)

function matches(song, query) {
  return !query || `${song.title} ${song.artist} ${song.lyrics}`.toLocaleLowerCase().includes(query)
}

function showToast(message) {
  toastMessage.value = message
  clearTimeout(toastTimer)
  toastTimer = setTimeout(() => { toastMessage.value = '' }, 2300)
}

function saveState() {
  try {
    localStorage.setItem(STORAGE_KEY, JSON.stringify({ version: 1, songs: songs.value, queue: queue.value }))
  } catch {
    showToast('Could not save in this browser. Export a JSON backup to keep your changes.')
  }
}

watch([songs, queue], saveState, { deep: true })

onMounted(() => {
  try {
    const saved = JSON.parse(localStorage.getItem(STORAGE_KEY) || 'null')
    if (saved && Array.isArray(saved.songs)) {
      songs.value = saved.songs
      queue.value = Array.isArray(saved.queue) ? saved.queue : []
    } else {
      // Import the browser data used by the earlier standalone HTML app.
      songs.value = JSON.parse(localStorage.getItem('songbook-studio-songs-v1') || '[]')
      queue.value = JSON.parse(localStorage.getItem('songbook-studio-queue-v1') || '[]')
    }
    queue.value = queue.value.filter(id => songs.value.some(song => song.id === id))
  } catch {
    songs.value = []
    queue.value = []
    showToast('Saved browser data could not be read.')
  }
  window.addEventListener('afterprint', clearPrintStyles)
})

onUnmounted(() => {
  clearTimeout(toastTimer)
  window.removeEventListener('afterprint', clearPrintStyles)
})

function clearForm() {
  songId.value = ''
  title.value = ''
  artist.value = ''
  lyrics.value = ''
}

function editSong(song) {
  songId.value = song.id
  title.value = song.title
  artist.value = song.artist || ''
  lyrics.value = song.lyrics
  window.scrollTo({ top: 0, behavior: 'smooth' })
}

function saveSong() {
  const cleanTitle = title.value.trim()
  const cleanLyrics = lyrics.value.trim()
  if (!cleanTitle || !cleanLyrics) {
    showToast('Add a song title and lyrics first.')
    return
  }
  const existing = songs.value.find(song => song.id === songId.value)
  const entry = {
    id: existing?.id || crypto.randomUUID(),
    title: cleanTitle,
    artist: artist.value.trim(),
    lyrics: cleanLyrics,
    updated: Date.now()
  }
  if (existing) {
    songs.value = songs.value.map(song => song.id === existing.id ? entry : song)
    showToast('Song updated.')
  } else {
    songs.value = [entry, ...songs.value]
    showToast('Song saved.')
  }
  clearForm()
}

function addToBook(song) {
  if (queue.value.includes(song.id)) {
    showToast('That song is already in the songbook.')
    return
  }
  queue.value = [...queue.value, song.id]
  showToast('Added to songbook.')
}

function moveSong(index, direction) {
  const target = index + direction
  if (target < 0 || target >= queue.value.length) return
  const next = [...queue.value]
  ;[next[index], next[target]] = [next[target], next[index]]
  queue.value = next
}

function deleteSong() {
  songs.value = songs.value.filter(song => song.id !== deletingId.value)
  queue.value = queue.value.filter(id => id !== deletingId.value)
  if (songId.value === deletingId.value) clearForm()
  showDeleteDialog.value = false
  showToast('Song deleted.')
}

function downloadBlob(filename, contents, type) {
  const blob = new Blob([contents], { type })
  const url = URL.createObjectURL(blob)
  const link = document.createElement('a')
  link.href = url
  link.download = filename
  link.click()
  setTimeout(() => URL.revokeObjectURL(url), 1000)
}

function exportJson() {
  downloadBlob('songbook-backup.json', JSON.stringify({ version: 1, songs: songs.value, queue: queue.value }, null, 2), 'application/json')
}

function importJson() {
  jsonInput.value?.click()
}

async function readJsonFile(event) {
  const file = event.target.files?.[0]
  event.target.value = ''
  if (!file) return
  try {
    const parsed = JSON.parse(await file.text())
    if (!Array.isArray(parsed.songs)) throw new Error('This file does not contain a songs list.')
    songs.value = parsed.songs.filter(song => song && song.title && song.lyrics).map(song => ({
      id: String(song.id || crypto.randomUUID()),
      title: String(song.title),
      artist: String(song.artist || ''),
      lyrics: String(song.lyrics),
      updated: Number(song.updated || Date.now())
    }))
    const songIds = new Set(songs.value.map(song => song.id))
    queue.value = Array.isArray(parsed.queue) ? parsed.queue.map(String).filter(id => songIds.has(id)) : []
    showToast('JSON library imported.')
  } catch (error) {
    showToast(error.message || 'Could not import this JSON file.')
  }
}

function escapeHtml(value) {
  return String(value || '').replace(/[&<>"']/g, char => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' })[char])
}

function exportWord() {
  if (!queuedSongs.value.length) return showToast('Add at least one song to the songbook first.')
  const markup = queuedSongs.value.map(song => `<section class="song"><h1>${escapeHtml(song.title)}</h1><div class="artist">${escapeHtml(song.artist)}</div><div class="lyrics">${escapeHtml(song.lyrics)}</div></section>`).join('')
  const documentHtml = `<!doctype html><html><head><meta charset="utf-8"><style>@page{size:${paperSize.value};margin:.7in}body{font:12pt Arial,sans-serif;color:#222;column-count:2;column-gap:.35in;column-fill:auto;-webkit-text-size-adjust:none;text-size-adjust:none}h1{font-family:${headingFont.value};font-size:${headingSize.value}pt;margin:0 0 3pt}.song{margin:0 0 12pt}.artist{color:#777;margin:0 0 5pt;font-size:9pt}.lyrics{font-size:${fontSize.value}pt!important;line-height:1.3;white-space:pre-wrap;text-size-adjust:none}</style></head><body>${markup}</body></html>`
  downloadBlob('my-songbook.doc', '\ufeff' + documentHtml, 'application/msword')
  showToast('Word document downloaded.')
}

function exportPdf() {
  if (!queuedSongs.value.length) return showToast('Add at least one song to the songbook first.')
  document.documentElement.style.setProperty('--print-lyrics-size', `${fontSize.value}pt`)
  document.documentElement.style.setProperty('--print-heading-size', `${headingSize.value}pt`)
  document.documentElement.style.setProperty('--print-heading-font', headingFont.value)
  document.documentElement.style.setProperty('--print-paper', paperSize.value)
  window.print()
}

function clearPrintStyles() {
  document.documentElement.style.removeProperty('--print-lyrics-size')
  document.documentElement.style.removeProperty('--print-heading-size')
  document.documentElement.style.removeProperty('--print-heading-font')
  document.documentElement.style.removeProperty('--print-paper')
}
</script>

<template>
  <div class="app-shell">
    <header class="topbar">
      <a class="brand" href="#" aria-label="Songbook Studio home" @click.prevent="activeTab = 'library'">
        <span class="brandmark">♫</span><span><strong>Songbook Studio</strong><small>Your lyrics, neatly gathered</small></span>
      </a>
      <div class="top-right"><span><i class="statusdot"></i> Saved in this browser</span><strong>{{ songCountText }}</strong></div>
    </header>

    <main class="wrap">
      <section class="hero">
        <div><div class="eyebrow">A home for every song</div><h1>Build your songbook.</h1><p>Keep lyrics organized, make edits, and turn your set into a songbook.</p></div>
        <div class="badge">LOCAL SONG LIBRARY</div>
      </section>

      <div class="data-tools">
        <span>Your songs stay in this browser. Use JSON backup to move them to another browser.</span>
        <div><button class="text-button" @click="importJson">Import JSON</button><button class="text-button" @click="exportJson">Download JSON backup</button><input ref="jsonInput" class="visually-hidden" type="file" accept=".json,application/json" @change="readJsonFile"></div>
      </div>

      <nav class="tabs" aria-label="Songbook sections">
        <button class="tab" :class="{ active: activeTab === 'library' }" @click="activeTab = 'library'">Song library <span>{{ songs.length }}</span></button>
        <button class="tab" :class="{ active: activeTab === 'songbook' }" @click="activeTab = 'songbook'">Songbook builder <span>{{ queue.length }}</span></button>
      </nav>

      <section v-if="activeTab === 'library'" class="grid">
        <article class="card">
          <div class="cardhead"><div><h2>Your songs</h2><div class="sub">Search your library, then edit or add songs to a songbook.</div></div><button class="primary" @click="clearForm">＋ Add song</button></div>
          <div class="search"><input v-model="librarySearch" aria-label="Search songs" placeholder="Search song title, artist, or lyrics…"></div>
          <div class="songlist">
            <div v-for="song in filteredSongs" :key="song.id" class="songrow">
              <div class="songinitial">{{ song.title.charAt(0).toUpperCase() }}</div>
              <div class="rowinfo"><div class="rowtitle">{{ song.title }}</div><div class="rowmeta">{{ song.artist || 'No artist' }} · {{ song.lyrics.trim().split(/\s+/).filter(Boolean).length }} words</div></div>
              <div class="actions"><button class="iconbtn" title="Add to songbook" @click="addToBook(song)">＋</button><button class="iconbtn" title="Edit song" @click="editSong(song)">✎</button><button class="iconbtn" title="Delete song" @click="deletingId = song.id; showDeleteDialog = true">×</button></div>
            </div>
            <div v-if="!filteredSongs.length" class="empty">{{ songs.length ? 'No songs match your search.' : 'Your library is empty. Add a song to get started.' }}</div>
          </div>
        </article>

        <article class="card">
          <div class="cardhead"><div><h2>{{ songId ? 'Edit song' : 'Add a song' }}</h2><div class="sub">Title, artist, and lyrics stay together.</div></div></div>
          <form class="editor" @submit.prevent="saveSong">
            <div class="field"><label for="song-title">SONG TITLE</label><input id="song-title" v-model="title" placeholder="e.g. A quiet morning" required></div>
            <div class="field"><label for="song-artist">ARTIST / NOTES <span class="optional">· optional</span></label><input id="song-artist" v-model="artist" placeholder="Artist, writer, or key"></div>
            <div class="field"><label for="song-lyrics">LYRICS</label><textarea id="song-lyrics" v-model="lyrics" placeholder="Paste or write the lyrics here…" required></textarea></div>
            <div class="edit-actions"><span class="helper">Saved as JSON in your browser.</span><div><button v-if="songId" class="secondary" type="button" @click="clearForm">Cancel</button><button class="primary" type="submit">{{ songId ? 'Update song' : 'Save song' }}</button></div></div>
          </form>
        </article>
      </section>

      <section v-else class="builder-grid">
        <article class="card result-card">
          <div class="list-head"><div><h2>Your songbook</h2><div class="sub">Add songs from your library in the order you want.</div></div><button class="danger" @click="queue = []">Clear all</button></div>
          <div class="search builder-search"><input v-model="builderSearch" aria-label="Search songs to add" placeholder="Search songs to add…"></div>
          <div class="songlist builder-results">
            <div v-for="song in builderSongs" :key="song.id" class="songrow"><div class="rowinfo"><div class="rowtitle">{{ song.title }}</div><div class="rowmeta">{{ song.artist || 'No artist' }}</div></div><button class="iconbtn" title="Add song" @click="addToBook(song)">＋</button></div>
            <div v-if="!builderSongs.length" class="empty">No matching songs.</div>
          </div>
          <div class="queue">
            <div v-for="(song, index) in queuedSongs" :key="song.id" class="queue-item"><span class="number">{{ String(index + 1).padStart(2, '0') }}</span><span class="qname">{{ song.title }}</span><div class="qtools"><button class="mini" title="Move up" :disabled="index === 0" @click="moveSong(index, -1)">↑</button><button class="mini" title="Move down" :disabled="index === queuedSongs.length - 1" @click="moveSong(index, 1)">↓</button><button class="mini" title="Remove" @click="queue = queue.filter(id => id !== song.id)">×</button></div></div>
            <div v-if="!queuedSongs.length" class="empty">Your songbook is empty. Search above and add a song.</div>
          </div>
          <div class="export-options">
            <h3>Page & type settings</h3>
            <div class="controls">
              <label>Paper size<select v-model="paperSize"><option value="letter">US Letter</option><option value="A4">A4</option></select></label>
              <label>Lyrics size<select v-model.number="fontSize"><option :value="10">Compact · 10 pt</option><option :value="12">Comfortable · 12 pt</option><option :value="14">Large · 14 pt</option><option :value="16">Extra large · 16 pt</option></select></label>
              <label>Song heading style<select v-model="headingFont"><option v-for="font in headingFonts" :key="font.value" :value="font.value">{{ font.label }}</option></select></label>
              <label>Heading size<select v-model.number="headingSize"><option :value="10">Extra small · 10 pt</option><option :value="12">Small · 12 pt</option><option :value="14">Compact · 14 pt</option><option :value="16">Medium · 16 pt</option><option :value="18">Large · 18 pt</option><option :value="22">Extra large · 22 pt</option><option :value="26">Largest · 26 pt</option></select></label>
            </div>
            <div class="export-buttons"><button class="secondary" @click="exportWord">↓ Word document</button><button class="primary" @click="exportPdf">↓ Download PDF</button></div>
            <div class="helper export-help">Songs flow down the left column, then the right. For PDF, choose “Save as PDF” in the print window.</div>
          </div>
        </article>

        <section class="preview-wrap">
          <div class="previewtop"><span>PAGE PREVIEW</span><span>{{ queuedSongs.length }} SONG{{ queuedSongs.length === 1 ? '' : 'S' }}</span></div>
          <div class="preview">
            <div v-if="!queuedSongs.length" class="preview-empty">Add songs to your songbook to preview them.</div>
            <div v-else class="preview-sheet" :style="{ '--preview-lyrics-size': `${fontSize}pt`, '--preview-heading-size': `${headingSize}pt`, '--preview-heading-font': headingFont }">
              <article v-for="song in queuedSongs" :key="song.id" class="preview-song"><h2>{{ song.title }}</h2><div v-if="song.artist" class="artist">{{ song.artist }}</div><pre>{{ song.lyrics }}</pre></article>
            </div>
          </div>
        </section>
      </section>
    </main>

    <div v-if="showDeleteDialog" class="modalback" role="dialog" aria-modal="true" aria-labelledby="delete-title" @click.self="showDeleteDialog = false">
      <div class="modal"><h2 id="delete-title">Delete this song?</h2><p>This removes the song from your library and songbook.</p><div class="modalbuttons"><button class="secondary" @click="showDeleteDialog = false">Cancel</button><button class="danger" @click="deleteSong">Delete song</button></div></div>
    </div>
    <div v-if="toastMessage" class="toast" role="status">{{ toastMessage }}</div>

    <div class="printbook" aria-hidden="true">
      <article v-for="song in queuedSongs" :key="song.id" class="print-song"><h1>{{ song.title }}</h1><div v-if="song.artist" class="artist">{{ song.artist }}</div><pre>{{ song.lyrics }}</pre></article>
    </div>
  </div>
</template>
