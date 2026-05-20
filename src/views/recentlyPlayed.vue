<template>
  <div v-show="show">
    <div class="recently-played">
      <div class="title">
        <span>{{ $t('library.recentlyPlayed.title') }}</span>
        <button
          v-if="tracks.length > 0"
          class="clear-button"
          @click="clearHistory"
        >
          {{ $t('library.recentlyPlayed.clearAll') }}
        </button>
      </div>
      <div v-if="tracks.length === 0" class="empty">
        {{ $t('library.recentlyPlayed.empty') }}
      </div>
      <TrackList
        v-else
        :id="0"
        :tracks="tracks"
        type="playlist"
        :dbclick-track-func="'playAList'"
      />
      <div v-if="hasMore && tracks.length > 0" class="load-more">
        <button @click="loadMore">{{ $t('explore.loadMore') }}</button>
      </div>
    </div>
  </div>
</template>

<script>
import NProgress from 'nprogress';
import { mapState } from 'vuex';
import debounce from 'lodash/debounce';
import TrackList from '@/components/TrackList.vue';
import {
  getPlayHistory,
  getPlayHistoryCount,
  clearPlayHistory,
} from '@/utils/db';

export default {
  name: 'RecentlyPlayed',
  components: {
    TrackList,
  },
  data() {
    return {
      show: false,
      tracks: [],
      offset: 0,
      limit: 100,
      total: 0,
      hasMore: false,
    };
  },
  computed: {
    ...mapState(['player']),
    currentTrackID() {
      return this.player.currentTrackID;
    },
  },
  watch: {
    currentTrackID() {
      this.debouncedRefresh();
    },
  },
  created() {
    this.debouncedRefresh = debounce(() => {
      this.offset = 0;
      this.loadHistory();
    }, 300);
    setTimeout(() => {
      if (!this.show) NProgress.start();
    }, 1000);
    this.loadHistory();
  },
  activated() {
    this.offset = 0;
    this.loadHistory();
  },
  methods: {
    async loadHistory() {
      try {
        const count = await getPlayHistoryCount();
        this.total = count;
        const history = await getPlayHistory(this.limit, this.offset);
        this.tracks = history.map(item => ({
          id: item.trackId,
          name: item.trackName,
          ar: [{ name: item.artistName }],
          al: {
            name: item.albumName,
            picUrl: item.albumCover,
          },
          dt: item.duration,
        }));
        this.hasMore = this.offset + this.limit < this.total;
        NProgress.done();
        this.show = true;
      } catch (e) {
        console.error('[recentlyPlayed] loadHistory failed', e);
        NProgress.done();
        this.show = true;
      }
    },
    async loadMore() {
      this.offset += this.limit;
      try {
        const history = await getPlayHistory(this.limit, this.offset);
        const newTracks = history.map(item => ({
          id: item.trackId,
          name: item.trackName,
          ar: [{ name: item.artistName }],
          al: {
            name: item.albumName,
            picUrl: item.albumCover,
          },
          dt: item.duration,
        }));
        this.tracks = [...this.tracks, ...newTracks];
        this.hasMore = this.offset + this.limit < this.total;
      } catch (e) {
        console.error('[recentlyPlayed] loadMore failed', e);
      }
    },
    async clearHistory() {
      if (!confirm(this.$t('library.recentlyPlayed.clearAll') + '?')) return;
      await clearPlayHistory();
      this.tracks = [];
      this.offset = 0;
      this.total = 0;
      this.hasMore = false;
    },
  },
};
</script>

<style lang="scss" scoped>
.recently-played {
  margin-top: 128px;
  margin-bottom: 128px;

  .title {
    font-size: 2.4rem;
    font-weight: 700;
    color: var(--color-text);
    display: flex;
    align-items: center;
    justify-content: space-between;
    margin-bottom: 24px;

    .clear-button {
      font-size: 0.875rem;
      font-weight: 600;
      color: var(--color-text);
      opacity: 0.68;
      padding: 8px 16px;
      border-radius: 8px;
      background: var(--color-secondary-bg-for-transparent);
      border: none;
      cursor: pointer;
      transition: 0.2s;

      &:hover {
        opacity: 1;
        background: var(--color-primary-bg-for-transparent);
        color: var(--color-primary);
      }
    }
  }

  .empty {
    text-align: center;
    color: var(--color-text);
    opacity: 0.48;
    font-size: 1rem;
    margin-top: 80px;
  }

  .load-more {
    display: flex;
    justify-content: center;
    margin-top: 24px;

    button {
      font-size: 0.875rem;
      font-weight: 600;
      color: var(--color-text);
      opacity: 0.68;
      padding: 8px 16px;
      border-radius: 8px;
      background: var(--color-secondary-bg-for-transparent);
      border: none;
      cursor: pointer;
      transition: 0.2s;

      &:hover {
        opacity: 1;
        background: var(--color-primary-bg-for-transparent);
        color: var(--color-primary);
      }
    }
  }
}
</style>
