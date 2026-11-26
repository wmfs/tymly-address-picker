<!--
  Address picker for the cardscript Input.Address element (@wmfs/tymly-address-picker).

  Rendered by cardscript-to-quasar (lib/builders/Input.Address.js), which passes:
    config         resolved settings (see cardscript-to-quasar lib/utils/address-picker-config.js)
    model          the card data the picker reads and writes
    validation     v$.data from vuelidate
    invalidFields  fields that failed on submit, with messages
  Optional:
    execute        async ({ stateMachineName, input, sendResponse }) => ({ status, ctx }).
                   Defaults to this.$tymly.executions.execute (TymlySDK from @wmfs/tymly-cardscript-sdk).

  It never changes `model` directly: it emits `update` with the values to merge.

  States follow the "Tymly unified UI" design board: search, results, too many matches,
  no matches, search unavailable, error, address chosen, manual entry, and a phone layout.
-->
<template>
  <div
    class="tap"
    :class="{ 'tap--phone': isPhone, 'tap--error': hasError }"
  >
    <!-- Manual entry -->
    <template v-if="isManual">
      <a
        href="#"
        class="tap__link tap__back"
        @click.prevent="backToSearch"
      >‹ Back to address search</a>
      <div class="tap__heading q-mt-md">{{ config.manualEntryTitle }}</div>

      <div
        v-if="config.manualEntryNotice"
        class="tap__banner tap__banner--change q-mt-md"
      >
        {{ config.manualEntryNotice }}
      </div>

      <div class="tap__panel q-mt-md">
        <div
          v-for="field in config.manualEntryFields"
          :id="`CARDSCRIPT_${field.id}`"
          :key="field.id"
          class="tap__field"
          :class="{ 'tap__field--error': fieldError(field.id), 'tap__field--short': field.width === 'short' }"
        >
          <label
            :for="inputId(field.id)"
            class="tap__label"
          >
            {{ field.title }}
            <span
              v-if="!field.required"
              class="tap__optional"
            >(optional)</span>
          </label>
          <p
            v-if="field.hint"
            :id="inputId(field.id) + '-hint'"
            class="tap__hint"
          >{{ field.hint }}</p>
          <p
            v-if="fieldError(field.id)"
            :id="inputId(field.id) + '-error'"
            class="tap__error-message"
          >
            <span class="sr-only">Error: </span>{{ fieldError(field.id) }}
          </p>
          <input
            :id="inputId(field.id)"
            class="tap__input"
            :value="model[field.id]"
            :autocomplete="field.autocomplete || 'off'"
            :aria-invalid="fieldError(field.id) ? 'true' : 'false'"
            :aria-describedby="describedBy(field)"
            @input="e => update({ [field.id]: e.target.value || null })"
            @blur="touch(field.id)"
          >
        </div>

        <div
          v-if="config.showMap"
          class="tap__field"
        >
          <div class="tap__label">
            Location <span class="tap__optional">(optional)</span>
          </div>
          <p class="tap__hint">Drag the pin to the property, or use your current location.</p>
          <div class="tap__map">
            <q-map
              :key="`manual-${manualMapKey}`"
              :locked="false"
              :draggable="true"
              :centre-latitude="manualPin.latitude"
              :centre-longitude="manualPin.longitude"
            >
              <q-map-circle
                format="LatLon"
                :latitude="manualPin.latitude"
                :longitude="manualPin.longitude"
                :showMarker="true"
                label="Property"
                @latitude="v => update({ [keys.latitude]: v })"
                @longitude="v => update({ [keys.longitude]: v })"
              />
            </q-map>
          </div>
          <div
            v-if="config.allowGeoLocation"
            class="tap__row q-mt-sm"
          >
            <q-btn
              class="tap__quiet-btn"
              no-caps
              unelevated
              :loading="locating"
              label="Use my current location"
              @click="pinMyLocation"
            />
            <span
              v-if="hasManualPin"
              class="tap__muted"
            >Pin at {{ formatCoord(model[keys.latitude]) }}, {{ formatCoord(model[keys.longitude]) }}</span>
          </div>
          <p
            v-if="locationError"
            class="tap__warning q-mt-sm"
            role="status"
          >{{ locationError }}</p>
        </div>
      </div>
    </template>

    <!-- Address chosen -->
    <div
      v-else-if="isSelected"
      class="tap__panel"
    >
      <div class="tap__selected-head">
        <div class="tap__selected-text">
          <div class="tap__selected-line">{{ selected.line || selected.label }}</div>
          <div
            v-if="selected.locality"
            class="tap__muted"
          >{{ selected.locality }}</div>
        </div>
        <a
          href="#"
          class="tap__link tap__change"
          @click.prevent="changeAddress"
        >Change<span class="sr-only"> address</span></a>
      </div>

      <div class="tap__selected-body q-mt-md">
        <div
          v-if="config.showMap && selectedHasPoint"
          class="tap__map tap__map--small"
        >
          <q-map
            :key="`selected-${selected.uprn}`"
            :locked="true"
          >
            <q-map-circle
              format="LatLon"
              :latitude="selected.latitude"
              :longitude="selected.longitude"
              :showMarker="true"
              :locked="true"
              :label="selected.line || selected.label"
            />
          </q-map>
        </div>
        <dl class="tap__summary">
          <div>
            <dt>UPRN</dt>
            <dd class="tap__mono">{{ details && details.maskingNumber ? selected.uprn : (selected.reference || selected.uprn) }}</dd>
          </div>
          <div v-if="details && details.maskingNumber">
            <dt>Masking number</dt>
            <dd class="tap__mono">{{ details.maskingNumber }}</dd>
          </div>
          <div v-if="selected.dataSourceLabel">
            <dt>Data source</dt>
            <dd>
              <span
                class="tap__tag"
                :class="selected.temporary ? 'tap__tag--info' : 'tap__tag--grey'"
              >{{ selected.dataSourceLabel }}</span>
            </dd>
          </div>
          <div v-if="details && details.classification">
            <dt>Classification</dt>
            <dd>
              {{ details.classification.title }} <span class="tap__mono tap__muted">({{ details.classification.code }})</span>
              <span
                v-if="details.classification.path"
                class="tap__dd-sub"
              >{{ details.classification.path }}</span>
            </dd>
          </div>
          <div v-if="status">
            <dt>Status</dt>
            <dd><span class="tap__tag tap__tag--ok">{{ status }}</span></dd>
          </div>
          <div v-if="hierarchy && hierarchy.parentUprn">
            <dt>Hierarchy</dt>
            <dd>
              Part of {{ hierarchy.parentAddressLabel || 'a parent property' }}
              <span class="tap__dd-sub"><span class="tap__mono">UPRN {{ hierarchy.parentUprn }}</span> · {{ hierarchy.siblingCount }} {{ hierarchy.siblingCount === 1 ? 'property' : 'properties' }} under it</span>
            </dd>
          </div>
          <div v-else-if="hierarchy && hierarchy.childrenCount > 0">
            <dt>Hierarchy</dt>
            <dd>Parent of {{ hierarchy.childrenCount }} {{ hierarchy.childrenCount === 1 ? 'property' : 'properties' }}</dd>
          </div>
          <div v-if="loadingDetails">
            <dt class="sr-only">Details</dt>
            <dd
              class="tap__muted tap__small"
              role="status"
            >Loading property details…</dd>
          </div>
        </dl>
      </div>
    </div>

    <!-- Search -->
    <div
      v-else
      class="tap__panel"
    >
      <q-btn
        v-if="isPhone && config.allowGeoLocation"
        class="tap__quiet-btn full-width q-mb-md"
        no-caps
        unelevated
        :loading="locating"
        label="Use my current location"
        @click="searchNearMe"
      />

      <div
        class="tap__field"
        :class="{ 'tap__field--error': hasError }"
      >
        <label
          :for="inputId(config.id)"
          class="tap__label tap__label--large"
        >{{ isPhone && config.allowGeoLocation ? 'Or search for the address' : config.title }}</label>
        <p
          v-if="config.subtitle"
          :id="inputId(config.id) + '-hint'"
          class="tap__hint"
        >{{ config.subtitle }}</p>
        <p
          v-if="hasError"
          :id="inputId(config.id) + '-error'"
          class="tap__error-message"
        >
          <span class="sr-only">Error: </span>{{ errorMessage }}
        </p>
        <div class="tap__search">
          <input
            :id="inputId(config.id)"
            v-model="query"
            type="search"
            class="tap__input tap__search-input"
            autocomplete="off"
            :placeholder="config.placeholder"
            :aria-invalid="hasError ? 'true' : 'false'"
            :aria-describedby="searchDescribedBy"
            @input="onType"
            @keydown.enter.prevent="search(1)"
          >
          <q-btn
            class="tap__search-btn"
            no-caps
            unelevated
            :loading="searching"
            label="Search"
            @click="search(1)"
          />
        </div>
      </div>

      <div
        v-if="!isPhone && (config.allowGeoLocation || config.allowManualEntry) && !hasSearched"
        class="tap__row q-mt-md"
      >
        <q-btn
          v-if="config.allowGeoLocation"
          class="tap__quiet-btn"
          no-caps
          unelevated
          :loading="locating"
          label="Use my current location"
          @click="searchNearMe"
        />
        <a
          v-if="config.allowManualEntry"
          href="#"
          class="tap__link"
          @click.prevent="enterManually"
        >I can't find the address</a>
      </div>

      <p
        v-if="locationError"
        class="tap__warning q-mt-md"
        role="status"
      >{{ locationError }}</p>

      <!-- Search unavailable -->
      <div
        v-if="searchFailed"
        class="tap__banner tap__banner--change q-mt-md"
        role="status"
      >
        <b>Search is unavailable</b>
        <div>Try again in a moment<template v-if="config.allowManualEntry">, or enter the address by hand</template>.</div>
        <q-btn
          class="tap__quiet-btn q-mt-sm"
          no-caps
          unelevated
          label="Try again"
          @click="search(page)"
        />
      </div>

      <!-- Too many matches -->
      <div
        v-else-if="tooMany"
        class="tap__banner tap__banner--change q-mt-md"
        role="status"
      >
        <b>More than {{ config.tooManyResults }} addresses match "{{ lastQuery }}"</b>
        <div>Add more of the address, for example a full postcode, a street name or a house number.</div>
      </div>

      <!-- No matches -->
      <div
        v-else-if="hasSearched && totalHits === 0 && !searching"
        class="tap__banner tap__banner--change q-mt-md"
        role="status"
      >
        <b>No addresses match "{{ lastQuery }}"</b>
        <div>Check the spelling, or try a postcode, street name or UPRN.</div>
      </div>

      <!-- Results -->
      <template v-else-if="results.length > 0">
        <div class="tap__row tap__results-head q-mt-md">
          <p
            class="tap__count"
            role="status"
          >{{ resultsSummary }}</p>
          <a
            v-if="config.showMap && resultsWithPoints.length > 0"
            href="#"
            class="tap__link tap__small"
            :aria-expanded="showResultsMap ? 'true' : 'false'"
            @click.prevent="showResultsMap = !showResultsMap"
          >{{ showResultsMap ? 'Hide map' : 'Show on map' }}</a>
        </div>

        <div
          v-if="showResultsMap"
          class="tap__map q-mt-sm"
        >
          <q-map
            :key="`results-${page}-${lastQuery}`"
            :locked="true"
            :draggable="true"
          >
            <q-map-circle
              v-for="r in resultsWithPoints"
              :key="r.uprn"
              format="LatLon"
              :latitude="r.latitude"
              :longitude="r.longitude"
              :showMarker="true"
              :locked="true"
              :label="r.line || r.label"
            />
          </q-map>
        </div>

        <fieldset class="tap__fieldset q-mt-sm">
          <legend class="sr-only">Choose the address</legend>
          <div class="tap__list">
            <label
              v-for="r in results"
              :key="r.uprn"
              class="tap__option"
              :class="{ 'tap__option--selected': r.uprn === model[keys.uprn] }"
            >
              <input
                type="radio"
                :name="inputId(config.id) + '-choice'"
                :value="r.uprn"
                :checked="r.uprn === model[keys.uprn]"
                @change="choose(r)"
              >
              <span class="tap__option-text">
                <span class="tap__option-line">{{ r.line || r.label }}</span>
                <span class="tap__option-meta">
                  <template v-if="r.locality">{{ r.locality }} · </template><span class="tap__mono">UPRN {{ r.uprn }}</span>
                </span>
              </span>
              <span
                v-if="r.dataSourceLabel"
                class="tap__tag"
                :class="r.temporary ? 'tap__tag--info' : 'tap__tag--grey'"
                :title="r.dataSource"
              >{{ r.temporary && isPhone ? 'Temporary' : r.dataSourceLabel }}</span>
            </label>
          </div>
        </fieldset>

        <div
          v-if="totalPages > 1"
          class="tap__row q-mt-sm"
        >
          <span class="tap__muted tap__small">Showing {{ firstShown }} to {{ lastShown }} of {{ totalHits }}. Or add a house number to narrow it down.</span>
          <q-pagination
            :model-value="page"
            class="tap__pagination"
            :max="totalPages"
            :max-pages="isPhone ? 3 : 5"
            direction-links
            boundary-links
            :disable="searching"
            @update:model-value="p => search(p)"
          />
        </div>
      </template>

      <div
        v-if="config.allowManualEntry && (hasSearched || isPhone)"
        class="q-mt-md"
      >
        <a
          href="#"
          class="tap__link"
          @click.prevent="enterManually"
        >I can't find the address</a>
      </div>
    </div>
  </div>
</template>

<script>
import QMap from '@wmfs/quasar-map-mapbox'
import template from 'lodash.template'

// Centre of the West Midlands, used when there is nothing better for the manual entry map.
const DEFAULT_CENTRE = { latitude: 52.4862, longitude: -1.8904 }
const MIN_TYPED_LENGTH = 3
const TYPING_DELAY_MS = 500

export default {
  name: 'TymlyAddressPicker',
  components: {
    QMap,
    QMapCircle: QMap.components.QMapCircle
  },
  props: {
    config: { type: Object, required: true },
    model: { type: Object, required: true },
    validation: { type: Object, default: null },
    invalidFields: { type: Object, default: null },
    execute: { type: Function, default: null }
  },
  emits: ['update'],
  data () {
    return {
      query: '',
      lastQuery: '',
      searching: false,
      searchFailed: false,
      hasSearched: false,
      results: [],
      totalHits: 0,
      totalPages: 0,
      page: 1,
      nearMe: false,
      showResultsMap: false,
      changing: false,
      locating: false,
      locationError: null,
      manualMapKey: 0,
      typingTimer: null,
      searchCount: 0,
      loadingDetails: false
    }
  },
  computed: {
    keys () {
      return this.config.keys
    },
    isPhone () {
      return this.$q.screen.lt.sm
    },
    isManual () {
      return this.model[this.keys.cannotFindAddress] === true
    },
    selected () {
      const uprn = this.model[this.keys.uprn]
      if (uprn === null || uprn === undefined || uprn === '') return null
      const stored = this.model[this.keys.selectedAddress]
      if (stored && String(stored.uprn) === String(uprn)) return stored
      // A draft or prefilled card may only have the UPRN and label
      return { uprn: String(uprn), label: this.model[this.keys.addressLabel] || String(uprn) }
    },
    isSelected () {
      return this.selected !== null && !this.changing
    },
    details () {
      return (this.selected && this.selected.details) || null
    },
    hierarchy () {
      return this.details && this.details.hierarchy
    },
    status () {
      return (this.details && this.details.status) || (this.selected && this.selected.status) || null
    },
    selectedHasPoint () {
      return this.selected && isNumber(this.selected.latitude) && isNumber(this.selected.longitude)
    },
    tooMany () {
      return this.hasSearched && !this.nearMe && this.totalHits > this.config.tooManyResults
    },
    resultsWithPoints () {
      return this.results.filter(r => isNumber(r.latitude) && isNumber(r.longitude))
    },
    firstShown () {
      return (this.page - 1) * this.config.resultsPerPage + 1
    },
    lastShown () {
      return Math.min(this.page * this.config.resultsPerPage, this.totalHits)
    },
    resultsSummary () {
      const n = this.totalHits
      const what = `${n} address${n === 1 ? '' : 'es'}`
      if (this.nearMe) return `${what} near you`
      const postcode = this.results.length && this.results[0].postcode
      if (postcode && normalise(this.lastQuery) === normalise(postcode)) return `${what} in ${postcode}`
      return `${what} found`
    },
    hasError () {
      const v = this.validation && this.validation[this.config.id]
      return !!(v && v.$error)
    },
    errorMessage () {
      const invalid = this.invalidFields && this.invalidFields[this.config.id]
      if (invalid && Array.isArray(invalid.messages) && invalid.messages.length) return invalid.messages.join(' ')
      return this.config.validation.message
    },
    searchDescribedBy () {
      return [
        this.config.subtitle ? this.inputId(this.config.id) + '-hint' : null,
        this.hasError ? this.inputId(this.config.id) + '-error' : null
      ].filter(Boolean).join(' ') || null
    },
    hasManualPin () {
      return isNumber(this.model[this.keys.latitude]) && isNumber(this.model[this.keys.longitude])
    },
    manualPin () {
      if (this.hasManualPin) {
        return { latitude: this.model[this.keys.latitude], longitude: this.model[this.keys.longitude] }
      }
      const geo = this.model.$GEOLOCATION
      if (geo && isNumber(geo.lat) && isNumber(geo.lng)) return { latitude: geo.lat, longitude: geo.lng }
      return DEFAULT_CENTRE
    }
  },
  watch: {
    // Drafts and prefilled cards may have an address without details yet
    'selected.uprn': {
      handler () {
        if (this.selected && !this.details) this.loadDetails(this.selected)
      },
      immediate: true
    }
  },
  beforeUnmount () {
    clearTimeout(this.typingTimer)
  },
  methods: {
    inputId (id) {
      return `tap-${id}`
    },
    runExecution (options) {
      if (this.execute) return this.execute(options)
      if (this.$tymly && this.$tymly.executions) return this.$tymly.executions.execute(options)
      throw new Error('Address picker needs an execute prop or this.$tymly (TymlySDK)')
    },
    update (patch) {
      this.$emit('update', patch)
    },
    touch (id) {
      const v = this.validation && this.validation[id]
      if (v && typeof v.$touch === 'function') v.$touch()
    },
    fieldError (id) {
      const v = this.validation && this.validation[id]
      if (!v || !v.$error) return null
      const invalid = this.invalidFields && this.invalidFields[id]
      if (invalid && Array.isArray(invalid.messages) && invalid.messages.length) return invalid.messages.join(' ')
      const field = this.config.manualEntryFields.find(f => f.id === id)
      if (v.postcode && v.postcode.$invalid) return 'Enter a real postcode'
      return field ? (field.message || `Enter ${field.title.toLowerCase()}`) : 'Check this field'
    },
    describedBy (field) {
      return [
        field.hint ? this.inputId(field.id) + '-hint' : null,
        this.fieldError(field.id) ? this.inputId(field.id) + '-error' : null
      ].filter(Boolean).join(' ') || null
    },
    formatCoord (v) {
      return Number(v).toFixed(5)
    },
    onType () {
      clearTimeout(this.typingTimer)
      if (!this.config.searchAsYouType) return
      if (this.query.trim().length < MIN_TYPED_LENGTH) return
      this.typingTimer = setTimeout(() => this.search(1), TYPING_DELAY_MS)
    },
    async search (page = 1, { nearMe = false, position = null } = {}) {
      clearTimeout(this.typingTimer)

      const query = nearMe ? '' : this.query.trim()
      if (!nearMe && query.length === 0) return

      // Ignore replies to older searches if the user has carried on typing
      const searchId = ++this.searchCount

      this.searching = true
      this.searchFailed = false
      this.nearMe = nearMe

      const input = {
        filter: query,
        page,
        limit: this.config.resultsPerPage
      }
      if (this.config.searchFields) input.fields = this.config.searchFields
      if (nearMe && position) {
        input.useGeoLocation = true
        input.$GEOLOCATION = position
      }

      try {
        const res = await this.runExecution({
          stateMachineName: this.config.searchEndpoint,
          input,
          sendResponse: 'COMPLETE'
        })

        if (searchId !== this.searchCount) return

        if (res.status !== 'SUCCEEDED') throw new Error(`Search ended with status ${res.status}`)

        const ctx = res.ctx || {}
        this.results = Array.isArray(ctx.results) ? ctx.results : []
        this.totalHits = ctx.totalHits || 0
        this.totalPages = ctx.totalPages || 0
        this.page = ctx.page || page
        this.lastQuery = query
        this.hasSearched = true
      } catch (err) {
        if (searchId !== this.searchCount) return
        console.log('Address picker search failed', err)
        this.searchFailed = true
        this.hasSearched = true
        this.results = []
      } finally {
        if (searchId === this.searchCount) this.searching = false
      }
    },
    async getPosition () {
      this.locating = true
      this.locationError = null
      try {
        const position = await getCurrentPosition({ timeout: 10000 })
        return { lat: position.coords.latitude, lng: position.coords.longitude }
      } catch {
        this.locationError = 'Your location is not available. Check location is allowed in your browser settings.'
        return null
      } finally {
        this.locating = false
      }
    },
    async searchNearMe () {
      const position = await this.getPosition()
      if (!position) return
      this.query = ''
      await this.search(1, { nearMe: true, position })
    },
    async pinMyLocation () {
      const position = await this.getPosition()
      if (!position) return
      this.update({ [this.keys.latitude]: position.lat, [this.keys.longitude]: position.lng })
      this.manualMapKey++
    },
    async choose (result) {
      this.changing = false
      this.update({
        [this.keys.uprn]: result.uprn,
        [this.keys.addressLabel]: result.label,
        [this.keys.selectedAddress]: { ...result },
        [this.keys.cannotFindAddress]: false
      })
      this.touch(this.config.id)
      await Promise.all([
        this.loadDetails(result),
        this.runSelectionEndpoint(result)
      ])
    },
    async loadDetails (address) {
      const endpoint = this.config.detailsEndpoint
      if (!endpoint || !address || this.loadingDetails === address.uprn) return

      this.loadingDetails = address.uprn
      try {
        const res = await this.runExecution({
          stateMachineName: endpoint,
          input: { uprn: address.uprn },
          sendResponse: 'COMPLETE'
        })
        const details = res.status === 'SUCCEEDED' && res.ctx && res.ctx.found !== false
          ? {
              maskingNumber: res.ctx.maskingNumber || null,
              classification: res.ctx.classification || null,
              status: res.ctx.status || null,
              hierarchy: res.ctx.hierarchy || null
            }
          : null
        // Only keep them if the user hasn't picked something else meanwhile
        if (details && String(this.model[this.keys.uprn]) === String(address.uprn)) {
          const current = this.model[this.keys.selectedAddress] || address
          this.update({ [this.keys.selectedAddress]: { ...current, details } })
        }
      } catch (err) {
        console.log('Address picker could not load property details', endpoint, err)
      } finally {
        if (this.loadingDetails === address.uprn) this.loadingDetails = false
      }
    },
    async runSelectionEndpoint (result) {
      const endpoint = this.config.onSelectionEndpoint
      if (!endpoint || !endpoint.name) return

      let input = { ...this.model, uprn: result.uprn }
      if (endpoint.params) {
        input = {}
        for (const [key, value] of Object.entries(endpoint.params)) {
          input[key] = typeof value === 'string'
            ? template(value, { interpolate: /{{([\s\S]+?)}}/g })({ data: { ...this.model, [this.keys.uprn]: result.uprn } }) || null
            : value
        }
      }

      try {
        const res = await this.runExecution({
          stateMachineName: endpoint.name,
          input,
          sendResponse: 'COMPLETE'
        })
        if (res.status === 'SUCCEEDED' && res.ctx) this.update(res.ctx)
      } catch (err) {
        console.log('Address picker selection endpoint failed', endpoint.name, err)
      }
    },
    changeAddress () {
      this.changing = true
      if (!this.hasSearched && this.selected && this.selected.postcode) {
        this.query = this.selected.postcode
        this.search(1)
      }
      this.$nextTick(() => {
        const input = document.getElementById(this.inputId(this.config.id))
        if (input) input.focus()
      })
    },
    enterManually () {
      this.changing = false
      this.update({
        [this.keys.cannotFindAddress]: true,
        [this.keys.uprn]: null,
        [this.keys.addressLabel]: null,
        [this.keys.selectedAddress]: null
      })
    },
    backToSearch () {
      this.update({ [this.keys.cannotFindAddress]: false })
    }
  }
}

function getCurrentPosition (options) {
  return new Promise((resolve, reject) => {
    if (typeof navigator === 'undefined' || !navigator.geolocation) return reject(new Error('Geolocation is not supported'))
    navigator.geolocation.getCurrentPosition(resolve, reject, options)
  })
}

function isNumber (v) {
  return typeof v === 'number' && Number.isFinite(v)
}

function normalise (s) {
  return String(s || '').replace(/\s/g, '').toUpperCase()
}
</script>

<style scoped>
.tap {
  --tap-ink: #0b0c0c;
  --tap-ink-2: #505a5f;
  --tap-ink-3: #626a6e;
  --tap-line: #dedcd7;
  --tap-panel: #ffffff;
  --tap-link: #1d70b8;
  --tap-link-hover: #003078;
  --tap-accent: var(--q-primary);
  --tap-accent-soft: #e8f1f8;
  --tap-danger: #d4351c;
  --tap-danger-ink: #aa2a16;
  --tap-warn-soft: #fff4dc;
  --tap-warn: #8a5300;
  --tap-ok-soft: #e7f3ec;
  --tap-ok: #005a30;
  --tap-neutral-soft: #efeeea;
  --tap-focus: #ffdd00;
  color: var(--tap-ink);
}

.tap__panel {
  background: var(--tap-panel);
  padding: 24px;
  display: flex;
  flex-direction: column;
}
.tap--phone .tap__panel { padding: 16px; }

.tap__heading { font-size: 24px; font-weight: 700; line-height: 1.2; }

.tap__field + .tap__field { margin-top: 18px; }
.tap__field--error { border-left: 4px solid var(--tap-danger); padding-left: 16px; }
.tap__field--short .tap__input { max-width: 180px; }

.tap__label { display: block; font-weight: 700; font-size: 16px; }
.tap__label--large { font-size: 19px; }
.tap__optional { font-weight: 400; color: var(--tap-ink-2); }
.tap__hint { margin: 4px 0 8px; color: var(--tap-ink-2); }
.tap__error-message { margin: 0 0 8px; color: var(--tap-danger-ink); font-weight: 700; }
.tap__label + .tap__input { margin-top: 6px; }

.tap__input {
  width: 100%;
  height: 44px;
  box-sizing: border-box;
  border: 2px solid var(--tap-ink);
  border-radius: 4px;
  padding: 0 12px;
  font: inherit;
  font-size: 17px;
  background: var(--tap-panel);
  color: var(--tap-ink);
}
.tap__input:focus {
  outline: 3px solid var(--tap-focus);
  outline-offset: 0;
  box-shadow: inset 0 0 0 2px var(--tap-ink);
}
.tap__field--error .tap__input { border-color: var(--tap-danger); }

.tap__search { display: flex; }
.tap__search-input { flex: 1; min-width: 0; border-right: 0; border-radius: 4px 0 0 4px; }
.tap__search-btn {
  min-height: 44px;
  border: 2px solid var(--tap-ink);
  border-radius: 0 4px 4px 0;
  background: var(--tap-neutral-soft);
  color: var(--tap-ink);
  font-weight: 600;
}
.tap__field--error .tap__search-btn { border-color: var(--tap-danger); }

.tap__quiet-btn {
  background: var(--tap-neutral-soft);
  color: var(--tap-ink);
  font-weight: 600;
  min-height: 40px;
}

.tap__row { display: flex; flex-wrap: wrap; gap: 12px 20px; align-items: center; }
.tap__results-head { align-items: baseline; }
.tap__results-head .tap__link { margin-left: auto; }
.tap__count { margin: 0; font-weight: 700; }
.tap__muted { color: var(--tap-ink-2); }
.tap__small { font-size: 14px; }
.tap__mono { font-family: "JetBrains Mono", ui-monospace, SFMono-Regular, Menlo, monospace; }

.tap__link { color: var(--tap-link); text-decoration: underline; text-underline-offset: 3px; cursor: pointer; }
.tap__link:hover { color: var(--tap-link-hover); }
.tap__link:focus { outline: none; background: var(--tap-focus); color: var(--tap-ink); box-shadow: 0 4px 0 var(--tap-ink); text-decoration: none; }
.tap__back { font-size: 15px; }
.tap__change { margin-left: auto; font-weight: 600; }

.tap__banner {
  border-left: 5px solid var(--tap-warn);
  background: var(--tap-warn-soft);
  padding: 12px 16px;
}
.tap__banner b { display: block; margin-bottom: 2px; }
.tap__warning { margin: 0; color: var(--tap-warn); font-weight: 600; }

.tap__fieldset { border: 0; margin: 0; padding: 0; min-width: 0; }
.tap__list { border: 1px solid var(--tap-line); border-radius: 10px; overflow: hidden; }
.tap__option {
  display: flex;
  gap: 12px;
  align-items: flex-start;
  padding: 11px 14px;
  cursor: pointer;
  background: var(--tap-panel);
}
.tap__option + .tap__option { border-top: 1px solid var(--tap-line); }
.tap__option:hover { background: #fafaf8; }
.tap__option--selected,
.tap__option--selected:hover { background: var(--tap-accent-soft); box-shadow: inset 4px 0 0 var(--tap-accent); }
.tap__option input { width: 22px; height: 22px; margin: 1px 0 0; accent-color: var(--tap-ink); flex: none; }
.tap__option-text { min-width: 0; flex: 1; }
.tap__option-line { display: block; font-weight: 600; }
.tap__option-meta { display: block; color: var(--tap-ink-3); font-size: 13px; }

.tap__tag {
  display: inline-block;
  padding: 2px 8px;
  border-radius: 4px;
  font-size: 13px;
  font-weight: 600;
  white-space: nowrap;
}
.tap__tag--grey { background: var(--tap-neutral-soft); color: var(--tap-ink-2); }
.tap__tag--info { background: var(--tap-accent-soft); color: var(--tap-accent); }
.tap__tag--ok { background: var(--tap-ok-soft); color: var(--tap-ok); }

.tap__selected-head { display: flex; align-items: flex-start; gap: 12px; }
.tap__selected-text { min-width: 0; }
.tap__selected-line { font-weight: 700; font-size: 19px; margin-bottom: 2px; }
.tap__selected-body { display: flex; flex-wrap: wrap; gap: 16px; }
.tap__selected-body .tap__map { flex: 1 1 260px; }
.tap__summary { flex: 1 1 260px; margin: 0; }
.tap__summary > div {
  display: grid;
  grid-template-columns: 110px 1fr;
  gap: 8px;
  padding: 8px 0;
  border-bottom: 1px solid var(--tap-line);
}
.tap__summary dt { font-weight: 700; }
.tap__summary dd { margin: 0; }
.tap__dd-sub { display: block; color: var(--tap-ink-3); font-size: 13px; }

.tap__map { border-radius: 10px; overflow: hidden; }
.tap__map :deep(.q-card) { margin-bottom: 0; }

.tap__pagination { margin-left: auto; }
.tap--phone .tap__pagination { margin-left: 0; }

.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0 0 0 0);
  white-space: nowrap;
  border: 0;
}
</style>
