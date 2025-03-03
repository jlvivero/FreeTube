<template>
  <div>
    <ft-loader
      v-if="isLoading"
      :fullscreen="true"
    />
    <ft-card
      v-else
      class="card"
    >
      <h2>{{ $t("Most Popular") }}</h2>
      <ft-element-list
        :data="shownResults"
      />
    </ft-card>
    <ft-refresh-widget
      :disable-refresh="isLoading"
      :last-refresh-timestamp="lastPopularRefreshTimestamp"
      :title="$t('Most Popular')"
      @click="fetchPopularInfo"
    />
  </div>
</template>

<script setup>

import { computed, onBeforeUnmount, onMounted, ref, shallowRef } from 'vue'

import FtLoader from '../../components/ft-loader/ft-loader.vue'
import FtCard from '../../components/ft-card/ft-card.vue'
import FtElementList from '../../components/FtElementList/FtElementList.vue'
import FtRefreshWidget from '../../components/ft-refresh-widget/ft-refresh-widget.vue'
import store from '../../store/index'

import { invidiousAPICall } from '../../helpers/api/invidious'
import { copyToClipboard, getRelativeTimeFromDate, setPublishedTimestampsInvidious, showToast } from '../../helpers/utils'
import { useI18n } from '../../composables/use-i18n-polyfill'

const { t } = useI18n()

const isLoading = ref(false)

const lastPopularRefreshTimestamp = computed(() => {
  return getRelativeTimeFromDate(store.getters.getLastPopularRefreshTimestamp, true)
})

/** @type {import('vue').ComputedRef<Array | null>} */
const popularCache = computed(() => {
  return store.getters.getPopularCache
})

const shownResults = shallowRef(popularCache.value || [])

onMounted(() => {
  document.addEventListener('keydown', keyboardShortcutHandler)

  if (shownResults.value.length === 0) {
    fetchPopularInfo()
  }
})

onBeforeUnmount(() => {
  document.removeEventListener('keydown', keyboardShortcutHandler)
})

async function fetchPopularInfo() {
  const searchPayload = {
    resource: 'popular',
    id: '',
    params: {}
  }

  isLoading.value = true
  const result = await invidiousAPICall(searchPayload)
    .catch((err) => {
      const errorMessage = t('Invidious API Error (Click to copy)')
      showToast(`${errorMessage}: ${err}`, 10000, () => {
        copyToClipboard(err)
      })
      return undefined
    })

  if (!result) {
    isLoading.value = false
    return
  }

  const items = result.filter((item) => {
    return item.type === 'video' || item.type === 'shortVideo' || item.type === 'channel' || item.type === 'playlist'
  })

  setPublishedTimestampsInvidious(items.filter(item => item.type === 'video' || item.type === 'shortVideo'))
  store.commit('setLastPopularRefreshTimestamp', new Date())
  shownResults.value = items
  isLoading.value = false
  store.commit('setPopularCache', items)
}

/**
 * @param {KeyboardEvent} event the keyboard event
 */
function keyboardShortcutHandler(event) {
  if (event.ctrlKey || document.activeElement.classList.contains('ft-input')) {
    return
  }
  // Avoid handling events due to user holding a key (not released)
  // https://developer.mozilla.org/en-US/docs/Web/API/KeyboardEvent/repeat
  if (event.repeat) { return }

  switch (event.key) {
    case 'r':
    case 'R':
    case 'F5':
      if (!isLoading.value) {
        fetchPopularInfo()
      }
      break
  }
}

</script>
<style scoped src="./Popular.css" />
