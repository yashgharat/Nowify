<template>
  <div id="app">
    <div
      v-if="player.playing"
      class="now-playing"
      :class="getNowPlayingClass()"
    >
      <b-container fluid>
        <b-row>
          <b-col
            ><div class="now-playing__cover">
              <img
                :src="player.trackAlbum.image"
                :alt="player.trackTitle"
                class="now-playing__image"
              /></div
          ></b-col>
          <b-col
            ><div
              v-if="this.playerResponse.context"
              class="now-playing__context has-context"
            >
              <b-row class="">
                <img class="now-playing__code" :src="getContextObject().url" />
              </b-row>
              <b-row>
                <h3
                  class="mb-5 now-playing__context-name"
                  v-text="getContextObject().contextName.toLowerCase()"
                ></h3>
              </b-row>
            </div>
            <div v-else class="now-playing__context null-context">
              <b-row class="pb-5">
                <img class="mb-5" :src="getCodeUrl(player.trackUri)" />
              </b-row>
            </div>
            <b-row class="">
              <h1
                class="now-playing__details now-playing__track mt-4"
                v-text="player.trackTitle"
              ></h1>
            </b-row>
            <b-row class="">
              <h2
                class="now-playing__details now-playing__artists"
                v-text="getTrackArtists"
              ></h2> </b-row
          ></b-col>
        </b-row>
      </b-container>
    </div>
    <div v-else class="now-playing" :class="getNowPlayingClass()">
      <h1 class="now-playing__idle-heading">No music is playing 😔</h1>
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
      playerResponse: {},
      playerData: this.getEmptyPlayer(),
      colourPalette: '',
      swatches: [],
      spotifyCode: {},
      currentPlaylist: {}
    }
  },

  computed: {
    /**
     * Return a comma-separated list of track artists.
     * @return {String}
     */
    getTrackArtists() {
      return this.player.trackArtists.join(', ')
    }
  },

  mounted() {
    this.setDataInterval()
  },

  beforeDestroy() {
    clearInterval(this.pollPlaying)
  },

  methods: {
    /**
     * Make the network request to Spotify to
     * get the current played track.
     */
    async getNowPlaying() {
      let data = {}

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
         * Fetch error.
         */
        if (!response.ok) {
          throw new Error(`An error has occured: ${response.status}`)
        }

        /**
         * Spotify returns a 204 when no current device session is found.
         * The connection was successful but there's no content to return.
         */
        if (response.status === 204) {
          data = this.getEmptyPlayer()
          this.playerData = data

          this.$nextTick(() => {
            this.$emit('spotifyTrackUpdated', data)
          })

          return
        }

        data = await response.json()
        this.playerResponse = data
      } catch (error) {
        this.handleExpiredToken()

        data = this.getEmptyPlayer()
        this.playerData = data

        this.$nextTick(() => {
          this.$emit('spotifyTrackUpdated', data)
        })
      }
    },

    /**
     * Get the Now Playing element class.
     * @return {String}
     */
    getNowPlayingClass() {
      const playerClass = this.player.playing ? 'active' : 'idle'
      return `now-playing--${playerClass}`
    },

    /**
     * Construct the context of Now Playing
     * @return {Object}
     */
    getContextObject() {
      var context = this.playerResponse.context.type
      let contextName = ''
      let url = ''

      if (context === 'album') {
        this.spotifyCode.url = this.getCodeUrl(this.player.trackAlbum.uri)
        this.spotifyCode.contextName = this.player.trackAlbum.title
      } else if (context === 'artist') {
        this.spotifyCode.url = this.getCodeUrl(this.player.artistUri)
        this.spotifyCode.contextName = this.player.trackArtists[0]
      } else if (context === 'playlist') {
        this.spotifyCode.url = this.getCodeUrl(this.playerResponse.context.uri)
        this.getPlaylistName(this.playerResponse.context.href).then(() => {
          this.spotifyCode.contextName = this.currentPlaylist.name
        })
      }

      url = this.spotifyCode.url
      contextName = this.spotifyCode.contextName
      return { contextName, url }
    },

    /**
     * Construct the Spotify Code
     * @return {String}
     */
    getCodeUrl(uri) {
      const baseURL = 'https://scannables.scdn.co/uri/plain/png/'
      if (!uri) return
      let background = this.colourPalette.background
      if (!background) return
      else background = background.slice(1)

      let textHex = this.colourPalette.text
      if (!textHex) return

      let colorStr = textHex === '#000' ? 'black' : 'white'

      return `${baseURL}${background}/${colorStr}/450/${uri}`
    },

    /**
     * get Playlist name
     * @return {String}
     */
    async getPlaylistName(url) {
      let m = {}
      try {
        const response = await fetch(url, {
          headers: {
            Authorization: `Bearer ${this.auth.accessToken}`
          }
        })

        /**
         * Fetch error.
         */
        if (!response.ok) {
          throw new Error(`An error has occured: ${response.status}`)
        }

        /**
         * Spotify returns a 204 when no current device session is found.
         * The connection was successful but there's no content to return.
         */
        if (response.status === 204) {
          this.currentPlaylist.name = ''
          return
        }

        m = await response.json()
        this.currentPlaylist.name = m.name
        return
      } catch (error) {
        this.handleExpiredToken()

        console.log(error)
      }
    },

    /**
     * Get the colour palette from the album cover.
     */
    getAlbumColours() {
      /**
       * No image (rare).
       */
      if (!this.player.trackAlbum?.image) {
        return
      }

      /**
       * Run node-vibrant to get colours.
       */
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
        trackTitle: ''
      }
    },

    /**
     * Poll Spotify for data.
     */
    setDataInterval() {
      clearInterval(this.pollPlaying)
      this.pollPlaying = setInterval(() => {
        this.getNowPlaying()
      }, 2500)
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
        this.playerResponse.error?.status === 401 ||
        this.playerResponse.error?.status === 400
      ) {
        this.handleExpiredToken()

        return
      }

      /**
       * Player is active, but user has paused.
       */
      if (this.playerResponse.is_playing === false) {
        this.playerData = this.getEmptyPlayer()

        return
      }

      /**
       * The newly fetched track is the same as our stored
       * one, we don't want to update the DOM yet.
       */
      if (this.playerResponse.item?.id === this.playerData.trackId) {
        return
      }

      /**
       * Store the current active track.
       */
      this.playerData = {
        playing: this.playerResponse.is_playing,
        trackArtists: this.playerResponse.item.artists.map(
          artist => artist.name
        ),
        artistUri: this.playerResponse.item.artists[0].uri,
        trackUri: this.playerResponse.item.uri,
        trackTitle: this.playerResponse.item.name,
        trackId: this.playerResponse.item.id,
        trackAlbum: {
          title: this.playerResponse.item.album.name,
          image: this.playerResponse.item.album.images[0].url,
          uri: this.playerResponse.item.uri
        }
      }
    },

    /**
     * Handle newly stored colour palette:
     * - Map data to readable format
     * - Get and store random colour combination.
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
     * Handle an expired access token from Spotify.
     */
    handleExpiredToken() {
      clearInterval(this.pollPlaying)
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
      })
    }
  }
}
</script>

<style src="@/styles/components/now-playing.scss" lang="scss" scoped></style>
