<template>
  <div class="px-1 px-sm-5 background noselect">
    <v-row no-gutters class="grey--text" align="center" justify="center">
      <v-col>
        <h1 style="font-size: 1.6em !important" class="subtitle-1 ml-2">
          {{ $t('modals.rss.title') }}
        </h1>
      </v-col>
      <v-col class="align-center justify-center">
        <v-card-actions class="justify-end">
          <v-btn small elevation="0" @click="close">
            <v-icon>{{ mdiClose }}</v-icon>
          </v-btn>
        </v-card-actions>
      </v-col>
    </v-row>

    <v-row class="ma-0 pa-0">
      <v-data-table
        id="articlesTable"
        :headers="headers"
        :items="filterUnread ? unreadArticles : articles"
        :items-per-page="15"
        item-key="key"
        :search="filter"
        :custom-filter="customFilter"
        multi-sort
        :sort-by="['parsedDate']"
        :sort-desc="[true]"
        :item-class="getRowStyle"
      >
        <template #top>
          <div class="mx-4 mb-5">
            <v-text-field v-model="filter" clearable :label="$t('filter')" />
            <v-row>
              <v-col>
                <v-checkbox class="my-0" v-model="filterUnread" :label="$t('modals.rss.filterRead')" hide-details />
              </v-col>
              <v-col>
                <v-btn style="float: right" small elevation="3" @click="markAllAsRead">
                  {{ $t('modals.rss.markAllAsRead') }}
                </v-btn>
              </v-col>
            </v-row>
          </div>
          <v-divider />
        </template>
        <template #[`item.title`]="{ item }">
          <a :href="item.link" target="_blank" v-text="item.title" />
        </template>
        <template #[`item.parsedDate`]="{ item }">
          {{ item.parsedDate.toLocaleString() }}
        </template>
        <template #[`item.actions`]="{ item }">
          <span class="rss-actions">
            <v-icon @click="markAsRead(item)">{{ mdiEmailOpen }}</v-icon>
            <v-icon @click="downloadTorrent(item)">{{ mdiDownload }}</v-icon>
          </span>
        </template>
      </v-data-table>
    </v-row>
  </div>
</template>

<script lang="ts">
import { General } from '@/mixins'
import { mapGetters, mapState } from 'vuex'
import { defineComponent } from 'vue'
import { FeedArticle } from '@/types/vuetorrent'
import { Feed, FeedRule } from '@/types/vuetorrent'
import { mdiClose, mdiDownload, mdiEmailOpen } from '@mdi/js'
import qbit from '@/services/qbit'

type RssState = { feeds: Feed[]; rules: FeedRule[] }

export default defineComponent({
  name: 'RssArticles',
  mixins: [General],
  data() {
    return {
      headers: [
        { text: this.$t('modals.rss.columnTitle.id'), value: 'id' },
        { text: this.$t('modals.rss.columnTitle.title'), value: 'title' },
        { text: this.$t('modals.rss.columnTitle.category'), value: 'category' },
        { text: this.$t('modals.rss.columnTitle.author'), value: 'author' },
        { text: this.$t('modals.rss.columnTitle.date'), value: 'parsedDate' },
        { text: this.$t('modals.rss.columnTitle.feedName'), value: 'feedName' },
        { text: this.$t('modals.rss.columnTitle.actions'), value: 'actions', sortable: false }
      ],
      filter: '',
      filterUnread: false,
      mdiEmailOpen,
      mdiDownload,
      mdiClose
    }
  },
  created() {
    this.$store.commit('FETCH_FEEDS')
  },
  mounted() {
    document.addEventListener('keydown', this.handleKeyboardShortcut)
  },
  beforeDestroy() {
    document.removeEventListener('keydown', this.handleKeyboardShortcut)
  },
  computed: {
    ...mapState(['rss']),
    ...mapGetters(['getModals']),
    articles(): FeedArticle[] {
      const articles: FeedArticle[] = []
      ;(this.rss as RssState).feeds.forEach((feed: Feed) => {
        feed.articles && articles.push(...feed.articles.map(article => ({ key: `${feed.uid}|${article.id}`, feedName: feed.name, parsedDate: new Date(article.date), ...article })))
      })
      return articles
    },
    unreadArticles() {
      return this.articles.filter(article => !article.isRead)
    }
  },
  methods: {
    close() {
      this.$router.back()
    },
    customFilter(value: string, query: string, item?: any): boolean {
      const article = item as FeedArticle
      return article.title.toLowerCase().indexOf(query.toLowerCase()) !== -1
    },
    getRowStyle(item: FeedArticle) {
      return item.isRead ? 'rss-read' : 'rss-unread'
    },
    downloadTorrent(item: FeedArticle) {
      this.createModal('AddModal', { initialMagnet: item.torrentURL })
    },
    async markAsRead(item: FeedArticle) {
      await qbit.markAsRead(item.feedName, item.id)
      this.$store.commit('FETCH_FEEDS')
    },
    async markAllAsRead() {
      for (const article of this.unreadArticles) {
        await qbit.markAsRead(article.feedName, article.id)
      }
      this.$store.commit('FETCH_FEEDS')
    },
    handleKeyboardShortcut(e: KeyboardEvent) {
      if (e.key === 'Escape' && this.getModals().length === 0) {
        this.close()
      }
    }
  }
})
</script>

<style scoped>
.rss-actions {
  display: flex;
  flex-direction: row;
}
</style>

<style>
.rss-unread {
  color: #ccc;
}
.rss-read {
  color: grey;
}
</style>
