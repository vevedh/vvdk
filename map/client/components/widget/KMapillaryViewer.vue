<template>
  <div id="mapillary-widget" class="column">
    <q-resize-observer @resize="onResized" />
    <div
      class="col"
      id="mapillary-container"
    />
  </div>
</template>

<script>
import _ from 'lodash'
import L from 'leaflet'
import logger from 'loglevel'
import { ref } from 'vue'
import { Viewer } from 'mapillary-js'
import 'mapillary-js/dist/mapillary.css'
import distance from '@turf/distance'
import { point } from '@turf/helpers'
import { KPanel } from '../../../../core/client/components'
import { useCurrentActivity, useHighlight } from '../../composables'

export default {
  components: {
    KPanel
  },
  props: {
    highlight: {
      type: Object,
      default: () => ({ zOrder: 1 })
    }
  },
  computed: {
    location () {
      return this.hasSelectedLocation() && this.getSelectedLocation()
    }
  },
  watch: {
    location: {
      handler () {
        this.refresh()
      }
    }
  },
  methods: {
    saveStates () {
      this.selection.mapillary = {
        location: this.location,
        bearing: this.bearing,
        imageId: this.imageId
      }
    },
    restoreStates () {
      const states = this.selection.mapillary
      this.location = states.location
      this.bearing = states.bearing
      this.imageId = states.imageId
    },
    getMarkerFeature () {
      const lat = this.location ? this.location.lat : 0
      const lon = this.location ? this.location.lng : 0
      const bearing = 225 + this.bearing || 0
      return {
        type: 'Feature',
        properties: {
          'icon-html': `<img style="${L.DomUtil.TRANSFORM}: translateX(-20px) translateY(-20px) rotateZ(${bearing}deg); width: 40p; height: 40px;" src="/icons/kdk/mapillary-marker.png">`
        },
        geometry: {
          type: 'Point',
          coordinates: [lon, lat]
        }
      }
    },
    async refresh () {
      this.hasImage = false
      if (_.has(this.selection, 'mapillary')) {
        this.restoreStates()
        if (this.imageId) {
          this.hasImage = true
          await this.refreshView()
        } else if (this.location) await this.moveCloseTo(this.location.lat, this.location.lng)
      } else if (this.hasSelectedItem()) {
        const location = this.getSelectedLocation()
        if (location) await this.moveCloseTo(location.lat, location.lng)
      }
    },
    async moveCloseTo (lat, lon) {
      // Query the images according a bbox that contains the given position
      const buffer = 0.0002 // around 25m
      const left = lon - buffer
      const right = lon + buffer
      const top = lat + buffer
      const bottom = lat - buffer
      const token = this.kActivity.mapillaryToken
      const query = `https://graph.mapillary.com/images?fields=id,computed_geometry&bbox=${left},${bottom},${right},${top}&access_token=${token}&limit=50`
      const response = await fetch(query)
      if (response.status !== 200) {
        this.hasImage = false
        throw new Error(`Impossible to fetch ${query}: ` + response.status)
      }
      const data = await response.json()
      const imageCount = data.data.length
      // If any results then search for the nearest image
      if (imageCount > 0) {
        const clickedPosition = point([lon, lat])
        let minDist
        _.forEach(data.data, image => {
          const dist = distance(clickedPosition, image.computed_geometry)
          if (!minDist || dist < minDist) {
            minDist = dist
            this.imageId = image.id
          }
        })
        this.hasImage = true
        this.refreshView()
      } else {
        this.$notify({ type: 'negative', message: this.$t('KMapillaryViewer.NO_IMAGE_FOUND_CLOSE_TO') })
      }
    },
    centerMap () {
      if (this.location) this.kActivity.center(this.location.lng, this.location.lat)
    },
    async refreshView () {
      this.highlight(this.getMarkerFeature())
      try {
        await this.mapillaryViewer.moveTo(this.imageId)
      } catch (error) {
        logger.error(error)
      }
    },
    onCurrentTimeChanged (time) {
      this.setupViewerFilters()
    },
    async onImageEvent (viewerImageEvent) {
      const image = viewerImageEvent.image
      this.imageId = image.id
      this.hasImage = true
      this.location = image.lngLat
      this.bearing = await this.mapillaryViewer.getBearing()
      this.centerMap()
      this.highlight(this.getMarkerFeature())
    },
    onResized (size) {
      if (this.mapillaryViewer) this.mapillaryViewer.resize()
    }
  },
  mounted () {
    // Create the viewer
    this.mapillaryViewer = new Viewer({
      container: 'mapillary-container',
      accessToken: this.kActivity.mapillaryToken
    })
    // Add event listeners
    this.mapillaryViewer.on('image', this.onImageEvent)
    // Configure the viewer
    this.location = undefined
    this.bearing = undefined
    this.key = undefined
    this.refresh()
  },
  async beforeUnmount () {
    // Remove event listeners
    this.mapillaryViewer.off('image', this.onImageEvent)
    // Save the states
    this.saveStates()
  },
  setup (props) {
    // Data
    const hasImage = ref(false)
    // Expose
    return {
      ...useCurrentActivity(),
      ...useHighlight('mapillary', props.highlight),
      hasImage
    }
  }
}
</script>
