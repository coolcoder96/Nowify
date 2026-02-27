<template>
  <div id="app">
    <div class="htdj-player" :class="getNowPlayingClass()">
      <div class="htdj-player__bg-track"></div>
      <div class="htdj-player__bg-progress" :style="progressOverlayStyle"></div>

      <div class="htdj-player__cover-stage">
        <template v-if="player.playing">
          <div
            class="htdj-player__cover"
            :key="`cover-${player.trackId}-${morphSeed}`"
          >
            <img
              :src="player.trackAlbum.image"
              :alt="player.trackTitle"
              class="htdj-player__image"
            />
          </div>
          <div
            class="htdj-player__details"
            :key="`details-${player.trackId}-${morphSeed}`"
          >
            <h1 class="htdj-player__track" v-text="player.trackTitle"></h1>
            <h2 class="htdj-player__artists" v-text="getTrackArtists"></h2>
          </div>
        </template>
        <div v-else class="htdj-player__idle">
          <h1 class="htdj-player__idle-heading">{{ idleMessage }}</h1>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import * as Vibrant from 'node-vibrant'

import props from '@/utils/props.js'

export default {
  name: 'NowPlaying',

  props: {
    auth: props.auth,
    endpoints: props.endpoints,
    player: props.player
  },

  data() {
    return {
      pollPlaying: '',
      progressTimer: '',
      playerResponse: {},
      playerData: this.getEmptyPlayer(),
      colourPalette: '',
      swatches: [],
      progressPercent: 0,
      morphSeed: 0,
      idleMessage: 'No music is playing right now.',
      pollingIntervalMs: 5000
    }
  },

  computed: {
    /**
     * Return a comma-separated list of track artists.
     * @return {String}
     */
    getTrackArtists() {
      return this.player.trackArtists.join(', ')
    },

    /**
     * Return styles to animate dark background layer
     * across the screen as the song progresses.
     * @return {Object}
     */
    progressOverlayStyle() {
      return {
        transform: `translateX(${this.progressPercent}%)`
      }
    }
  },

  mounted() {
    this.setDataInterval()
  },

  beforeDestroy() {
    clearInterval(this.pollPlaying)
    clearInterval(this.progressTimer)
  },

  methods: {
    /**
     * Make the network request to Spotify to
     * get the current played track.
     */
    async getNowPlaying() {
      try {
        const response = await fetch(
          `${this.endpoints.base}/${this.endpoints.nowPlaying}`,
          {
            headers: {
              Authorization: `Bearer ${this.auth.accessToken}`
            }
          }
        )

        /**
         * Spotify returns a 204 when no current device session is found.
         * The connection was successful but there's no content to return.
         */
        if (response.status === 204) {
          this.handleNoActiveSession()
          return
        }

        if (response.status === 429) {
          this.handleRateLimited(response)
          return
        }

        if (response.status === 403) {
          this.handlePlaybackUnavailable()
          return
        }

        if (response.status === 401 || response.status === 400) {
          this.handleExpiredToken()
          return
        }

        if (!response.ok) {
          this.handleNoActiveSession()
          return
        }

        this.idleMessage = 'No music is playing right now.'
        this.playerResponse = await response.json()
      } catch (error) {
        this.idleMessage = 'Waiting for Spotify connection…'
        this.playerData = this.getEmptyPlayer()
      }
    },

    /**
     * Get the Now Playing element class.
     * @return {String}
     */
    getNowPlayingClass() {
      const playerClass = this.player.playing ? 'active' : 'idle'
      return `htdj-player--${playerClass}`
    },

    /**
     * Get the colour palette from the album cover.
     */
    getAlbumColours() {
      if (!this.player.trackAlbum?.image) {
        return
      }

      Vibrant.from(this.player.trackAlbum.image)
        .quality(1)
        .clearFilters()
        .getPalette()
        .then(palette => {
          this.handleAlbumPalette(palette)
        })
    },

    /**
     * Return a formatted empty object for an idle player.
     * @return {Object}
     */
    getEmptyPlayer() {
      return {
        playing: false,
        trackAlbum: {},
        trackArtists: [],
        trackId: '',
        trackTitle: '',
        trackDurationMs: 0,
        trackProgressMs: 0,
        trackStartedAt: 0
      }
    },

    /**
     * Poll Spotify for data.
     */
    setDataInterval() {
      clearInterval(this.pollPlaying)
      this.pollPlaying = setInterval(() => {
        this.getNowPlaying()
      }, this.pollingIntervalMs)
    },

    /**
     * Keep song progress in sync and animate the overlay.
     */
    setProgressInterval() {
      clearInterval(this.progressTimer)

      if (!this.playerData.playing || !this.playerData.trackDurationMs) {
        this.progressPercent = 0
        return
      }

      const updateProgress = () => {
        const elapsed = Date.now() - this.playerData.trackStartedAt
        const rawProgress = (elapsed / this.playerData.trackDurationMs) * 100
        this.progressPercent = Math.max(0, Math.min(rawProgress, 100))
      }

      updateProgress()
      this.progressTimer = setInterval(updateProgress, 1000)
    },

    /**
     * Set the stylings of the app based on received colours.
     */
    setAppColours() {
      document.documentElement.style.setProperty(
        '--color-text-primary',
        this.colourPalette.text
      )

      document.documentElement.style.setProperty(
        '--colour-background-now-playing',
        this.colourPalette.background
      )
    },

    /**
     * Handle newly updated Spotify Tracks.
     */
    handleNowPlaying() {
      if (
        this.playerResponse.is_playing === false ||
        !this.playerResponse.item?.id
      ) {
        this.handleNoActiveSession()
        return
      }

      if (this.playerResponse.item.id === this.playerData.trackId) {
        this.playerData = {
          ...this.playerData,
          trackProgressMs: this.playerResponse.progress_ms,
          trackStartedAt: Date.now() - this.playerResponse.progress_ms
        }
        return
      }

      this.playerData = {
        playing: this.playerResponse.is_playing,
        trackArtists: this.playerResponse.item.artists.map(
          artist => artist.name
        ),
        trackTitle: this.playerResponse.item.name,
        trackId: this.playerResponse.item.id,
        trackDurationMs: this.playerResponse.item.duration_ms,
        trackProgressMs: this.playerResponse.progress_ms,
        trackStartedAt: Date.now() - this.playerResponse.progress_ms,
        trackAlbum: {
          title: this.playerResponse.item.album.name,
          image: this.playerResponse.item.album.images[0].url
        }
      }

      this.pollingIntervalMs = 5000
      this.setDataInterval()
      this.morphSeed += 1
    },

    /**
     * Handle newly stored colour palette.
     */
    handleAlbumPalette(palette) {
      let albumColours = Object.keys(palette)
        .filter(item => {
          return item === null ? null : item
        })
        .map(colour => {
          return {
            text: palette[colour].getTitleTextColor(),
            background: palette[colour].getHex()
          }
        })

      this.swatches = albumColours
      this.colourPalette =
        albumColours[Math.floor(Math.random() * albumColours.length)]

      this.$nextTick(() => {
        this.setAppColours()
      })
    },

    /**
     * 204 or no playable item.
     */
    handleNoActiveSession() {
      this.idleMessage = 'No music is playing right now.'
      this.playerData = this.getEmptyPlayer()
    },

    /**
     * 429 response: back off polling according to Retry-After.
     */
    handleRateLimited(response) {
      const retryAfter = Number(response.headers.get('Retry-After')) || 15
      this.idleMessage = `Spotify rate limit reached. Retrying in ${retryAfter}s…`
      this.playerData = this.getEmptyPlayer()
      this.pollingIntervalMs = Math.max(retryAfter * 1000, 5000)
      this.setDataInterval()
    },

    /**
     * 403 response: playback endpoints may be restricted.
     */
    handlePlaybackUnavailable() {
      this.idleMessage =
        'Playback status is unavailable for this account/app under Spotify API limits.'
      this.playerData = this.getEmptyPlayer()
      this.pollingIntervalMs = 30000
      this.setDataInterval()
    },

    /**
     * Handle an expired access token from Spotify.
     */
    handleExpiredToken() {
      clearInterval(this.pollPlaying)
      clearInterval(this.progressTimer)
      this.$emit('requestRefreshToken')
    }
  },

  watch: {
    /**
     * Watch the auth object returned from Spotify.
     */
    auth: function(oldVal, newVal) {
      if (newVal.status === false) {
        clearInterval(this.pollPlaying)
        clearInterval(this.progressTimer)
      }
    },

    /**
     * Watch the returned track object.
     */
    playerResponse: function() {
      this.handleNowPlaying()
    },

    /**
     * Watch our locally stored track data.
     */
    playerData: function() {
      this.$emit('spotifyTrackUpdated', this.playerData)

      this.$nextTick(() => {
        this.getAlbumColours()
        this.setProgressInterval()
      })
    }
  }
}
</script>

<style src="@/styles/components/now-playing.scss" lang="scss" scoped></style>
