<template>
  <div class="wrap" theme="light">
    <header>
      <h1>Max Player</h1>
      <div class="small muted">انسخ الـ iframe بعد إعداد الروابط</div>
    </header>

    <div class="player-area">
      <div id="player" class="media" ref="player">
        <div class="muted" v-if="!playlist.length">
          لا يوجد رابط — أضف رابط أو استعمل query param `links`
        </div>
      </div>

      <div class="">
        <input
          v-model="url"
          class="input grow"
          placeholder="ألصق رابط (YouTube/Vimeo/MP4/MP3/PDF/ أي رابط)"
        />
        <input
          v-model="label"
          class="input"
          placeholder="تسمية (اختياري)"
          style="max-width: 200px"
        />
        <button @click="onAdd" class="input">أضف</button>
        <button @click="onCopyIframe" class="input copy-btn">انسخ iframe</button>
      </div>

      <div class="note">
        يدعم: YouTube, Vimeo, ملفات الفيديو (mp4, webm), صوت (mp3,wav), PDF و صفحات iframe عامة.
        يمكن إضافة روابط متعددة.
      </div>

      <div class="playlist" id="playlist">
        <div class="item" v-for="(it, idx) in playlist" :key="idx">
          <div>
            <b>{{ it.label || 'Item ' + (idx + 1) }}</b>
            <div class="muted small">{{ it.url }}</div>
          </div>
          <div class="right">
            <button class="input small" @click="playIndex(idx)">تشغيل</button>
            <button class="input small" @click="openInNew(it.url)">فتح</button>
            <button class="input small" @click="remove(idx)">حذف</button>
          </div>
        </div>
      </div>

      <div style="margin-top: 12px; display: flex; gap: 8px; align-items: center; flex-wrap: wrap">
        <div class="muted small">مثال للمشاركة في iframe بعد استضافة هذا الملف:</div>
        <div
          id="example"
          class="small"
          style="
            background: rgba(255, 255, 255, 0.02);
            padding: 6px;
            border-radius: 6px;
            word-break: break-all;
          "
        >
          {{ exampleIframe }}
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'UniversalEmbedPlayer',
  data() {
    return {
      url: '',
      label: '',
      playlist: [],
      currentIndex: -1,
      hlsScriptLoaded: false,
    }
  },
  computed: {
    exampleIframe() {
      const base = location.href.split('#')[0].split('?')[0]
      const data = JSON.stringify(this.playlist)
      // base64 safe encoding for unicode
      const encoded = btoa(unescape(encodeURIComponent(data)))
      const src = base + '?links=' + encodeURIComponent(encoded)
      return `<iframe src="${src}" width="800" height="520" frameborder="0" allowfullscreen></iframe>`
    },
  },
  methods: {
    // Helpers
    isYouTube(url) {
      return /(?:youtube\.com\/watch\?|youtu\.be\/)/i.test(url)
    },
    youtubeEmbed(url) {
      let id = null
      const m = url.match(/[?&]v=([^&]+)/)
      if (m) id = m[1]
      else {
        const s = url.match(/youtu\.be\/([^?&]+)/)
        if (s) id = s[1]
      }
      return id ? `https://www.youtube.com/embed/${id}?rel=0&autoplay=0&modestbranding=1` : null
    },
    isVimeo(url) {
      return /vimeo\.com\/(?:video\/)?\d+/i.test(url)
    },
    vimeoEmbed(url) {
      const m = url.match(/(\d+)/)
      return m ? `https://player.vimeo.com/video/${m[1]}` : null
    },
    ext(url) {
      try {
        return url.split('?')[0].split('.').pop().toLowerCase()
      } catch (e) {
        return ''
      }
    },
    detectType(url) {
      try {
        const e = this.ext(url)
        if (this.isYouTube(url)) return 'youtube'
        if (this.isVimeo(url)) return 'vimeo'
        if (e === 'mp4' || e === 'webm' || e === 'ogg') return 'video'
        if (e === 'mp3' || e === 'wav' || e === 'm4a') return 'audio'
        if (e === 'pdf') return 'pdf'
        if (url.includes('.m3u8')) return 'hls'
        return 'iframe'
      } catch (e) {
        return 'iframe'
      }
    },

    // Player rendering
    clearPlayer() {
      const root = this.$refs.player || document.getElementById('player')
      root.innerHTML = ''
      return root
    },

    renderPlayerItem(item) {
      const root = this.clearPlayer()
      const type = this.detectType(item.url)

      if (type === 'youtube') {
        const src = this.youtubeEmbed(item.url)
        const ifr = document.createElement('iframe')
        ifr.src = src
        ifr.allow =
          'accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share'
        ifr.style.width = '100%'
        ifr.style.height = '100%'
        root.appendChild(ifr)
        return
      }

      if (type === 'vimeo') {
        const ifr = document.createElement('iframe')
        ifr.src = this.vimeoEmbed(item.url)
        ifr.style.width = '100%'
        ifr.style.height = '100%'
        root.appendChild(ifr)
        return
      }

      if (type === 'video') {
        const vid = document.createElement('video')
        vid.controls = true
        vid.src = item.url
        vid.setAttribute('playsinline', '')
        vid.style.width = '100%'
        vid.style.height = '100%'
        root.appendChild(vid)
        return
      }

      if (type === 'audio') {
        const audio = document.createElement('audio')
        audio.controls = true
        audio.src = item.url
        audio.style.width = '100%'
        root.appendChild(audio)
        return
      }

      if (type === 'pdf') {
        const ifr = document.createElement('iframe')
        ifr.src = item.url
        ifr.style.width = '100%'
        ifr.style.height = '100%'
        root.appendChild(ifr)
        return
      }

      if (type === 'hls') {
        const vid = document.createElement('video')
        vid.controls = true
        vid.style.width = '100%'
        vid.style.height = '100%'
        root.appendChild(vid)

        if (vid.canPlayType('application/vnd.apple.mpegurl')) {
          vid.src = item.url
        } else {
          // load hls.js dynamically
          if (!this.hlsScriptLoaded) {
            const s = document.createElement('script')
            s.src = 'https://cdn.jsdelivr.net/npm/hls.js@1'
            s.onload = () => {
              this.hlsScriptLoaded = true
              if (window.Hls) {
                const hls = new window.Hls()
                hls.loadSource(item.url)
                hls.attachMedia(vid)
              }
            }
            document.body.appendChild(s)
          } else {
            if (window.Hls) {
              const hls = new window.Hls()
              hls.loadSource(item.url)
              hls.attachMedia(vid)
            }
          }
        }
        return
      }

      // fallback iframe
      const ifr = document.createElement('iframe')
      ifr.src = item.url
      ifr.style.width = '100%'
      ifr.style.height = '100%'
      root.appendChild(ifr)
    },

    // UI handlers
    onAdd() {
      const url = (this.url || '').trim()
      if (!url) return
      this.playlist.push({ url, label: this.label || `Item ${this.playlist.length + 1}` })
      if (this.playlist.length === 1) {
        this.currentIndex = 0
        this.renderPlayerItem(this.playlist[0])
      }
      this.url = ''
      this.label = ''
      this.updateExample() // update example display
    },
    playIndex(i) {
      this.currentIndex = i
      this.renderPlayerItem(this.playlist[i])
    },
    remove(i) {
      if (i < 0 || i >= this.playlist.length) return
      this.playlist.splice(i, 1)
      if (this.playlist.length === 0) {
        this.clearPlayer()
      } else {
        const idx = Math.min(this.currentIndex, this.playlist.length - 1)
        this.currentIndex = idx
        this.renderPlayerItem(this.playlist[idx])
      }
      this.updateExample()
    },
    openInNew(url) {
      window.open(url, '_blank')
    },

    // iframe share
    updateExample() {
      // reactive computed exampleIframe will change automatically
    },
    onCopyIframe() {
      const iframe = this.exampleIframe
      if (!navigator.clipboard) {
        // fallback
        window.prompt('انسخ الكود يدوياً:', iframe)
        return
      }
      navigator.clipboard
        .writeText(iframe)
        .then(() => {
          alert('تم نسخ الـ iframe')
        })
        .catch(() => {
          window.prompt('انسخ الكود يدوياً:', iframe)
        })
    },

    // parse links param
    loadFromQuery() {
      const params = new URLSearchParams(window.location.search)
      if (params.has('links')) {
        try {
          const raw = params.get('links')
          // decode base64 that was encoded via encodeURIComponent + btoa
          const decoded = decodeURIComponent(escape(atob(raw)))
          const arr = JSON.parse(decoded)
          if (Array.isArray(arr)) {
            this.playlist = arr
            if (this.playlist.length) {
              this.currentIndex = 0
              this.renderPlayerItem(this.playlist[0])
            }
          }
        } catch (e) {
          // attempt legacy decode (if not encoded same way)
          try {
            const arr = JSON.parse(atob(params.get('links')))
            if (Array.isArray(arr)) {
              this.playlist = arr
              if (this.playlist.length) {
                this.currentIndex = 0
                this.renderPlayerItem(this.playlist[0])
              }
            }
          } catch (e2) {
            console.warn('failed to parse links param', e, e2)
          }
        }
      } else if (params.has('src')) {
        this.playlist = [{ url: params.get('src'), label: params.get('label') || '' }]
        if (this.playlist.length) {
          this.currentIndex = 0
          this.renderPlayerItem(this.playlist[0])
        }
      }
    },
  },
  mounted() {
    // load from query if present
    this.loadFromQuery()
    // ensure initial example computed value updates
    this.updateExample()
  },
}
</script>

<style scoped>
:root {
  --bg: #0f1724;
  --card: #3b3c3d;
  --accent: #06b6d4;
  --muted: #94a3b8;
  --glass: rgba(255, 255, 255, 0.03);
}
html,
body {
  font-family: Inter, system-ui, Arial, sans-serif;
}
.wrap {
  max-width: 980px;
  margin: 28px auto;
  width: 80%;
  padding: 18px;
  align-self: center;
  background: black;
  border-radius: 12px;
  box-shadow: 0 8px 30px rgba(2, 6, 23, 0.6);
  color: #e6eef6;
}
header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 12px;
}
h1 {
  font-size: 18px;
  margin: 0;
}
.player-area {
  margin-top: 16px;
  background: var(--glass);
  padding: 10px;
  border-radius: 10px;
}
.media {
  width: 100%;
  height: 520px;
  background: #000;
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: hidden;
}
.media iframe,
.media video,
.media audio {
  width: 100%;
  height: 100%;
  border: 0;
}
.controls {
  display: flex;
  flex-flow: row, wrap;
  gap: 8px;
  background-color: red;
  align-items: center;
  margin-top: 10px;
}
.input,
button {
  padding: 8px 10px;
  border-radius: 8px;
  border: 1px solid rgba(255, 255, 255, 0.06);
  background: transparent;
  color: inherit;
}
.playlist {
  margin-top: 12px;
  display: flex;
  flex-direction: column;
  gap: 8px;
}
.item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 8px;
  border-radius: 8px;
  background: rgba(255, 255, 255, 0.02);
}
.item b {
  font-size: 13px;
}
.muted {
  color: var(--muted);
  font-size: 13px;
}
.small {
  font-size: 12px;
}
.right {
  display: flex;
  gap: 8px;
}
.copy-btn {
  cursor: pointer;
}
.note {
  margin-top: 10px;
  color: var(--muted);
  font-size: 13px;
}
.flex {
  display: flex;
  gap: 8px;
}
.grow {
  flex: 1;
}

.player-wrapper {
  width: 100%;
  background: #000;
  border-radius: 10px;
  overflow: hidden;
}
@media (max-width: 600px) {
  .player-wrapper iframe,
  .player-wrapper video,
  .player-wrapper audio {
    height: 240px !important;
  }
  .controls {
    display: flex;
    flex-flow: row, wrap;
  }
}
</style>
