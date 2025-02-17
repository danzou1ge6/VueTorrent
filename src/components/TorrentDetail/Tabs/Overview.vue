<template>
  <v-flex>
    <v-row>
      <v-col cols="12" md="6">
        <v-card flat>
          <v-card-title class="overflow-wrap">{{ torrent.name }}</v-card-title>
          <v-card-subtitle>
            <div v-for="commentPart in splitString(comment)" :key="commentPart">
              <a v-if="stringContainsUrl(commentPart)" target="_blank" :href="commentPart">{{ commentPart }}</a>
              <span v-else>{{ commentPart }}</span>
            </div>
            {{ torrent.hash }}
            <v-btn rounded @click="copyHash">{{ $t('rightClick.copy') }}</v-btn>
          </v-card-subtitle>
          <v-card-text>
            <v-row>
              <v-col cols="4" md="3">
                <v-progress-circular v-if="isFetchingMetadata" indeterminate :size="100" color="torrent-metadata">
                  {{ $t('modals.detail.pageOverview.fetchingMetadata') }}
                </v-progress-circular>
                <v-progress-circular v-else-if="torrent?.progress === 100" :size="100" :width="15" :value="100" color="torrent-seeding">
                  <v-icon color="torrent-seeding">{{ mdiCheck }}</v-icon>
                </v-progress-circular>
                <v-progress-circular v-else :rotate="-90" :size="100" :width="15" :value="torrent?.progress ?? 0" color="accent">
                  {{ (torrent.progress ?? 0) | progress }}
                </v-progress-circular>
              </v-col>
              <v-col cols="8" md="9" class="d-flex align-center justify-center flex-column">
                <div v-if="isFetchingMetadata">
                  <span>
                    {{ $t('modals.detail.pageOverview.waitingForMetadata') }}
                  </span>
                </div>
                <div v-else-if="shouldRenderPieceStates">
                  <canvas id="pieceStates" width="0" height="1" />
                </div>
                <div v-if="!isFetchingMetadata && !shouldRenderPieceStates">
                  <span>{{ $t('modals.detail.pageOverview.canvasRefreshDisabled') }}</span>
                </div>
                <div v-if="torrentPieceCount !== -1">
                  <span>{{ torrentPieceOwned }} / {{ torrentPieceCount }} ({{ pieceSize }})</span>
                </div>
                <div>
                  <v-icon>{{ mdiArrowDown }}</v-icon>
                  {{ torrent?.dlspeed | formatSpeed(shouldUseBitSpeed()) }}

                  <v-icon>{{ mdiArrowUp }}</v-icon>
                  {{ torrent?.upspeed | formatSpeed(shouldUseBitSpeed()) }}
                </div>
              </v-col>
            </v-row>
            <v-row>
              <v-col cols="6">
                <div>{{ $t('torrent.properties.save_path') }}:</div>
                <div>{{ torrent.savePath }}</div>
                <v-btn fab x-small color="accent" @click="changeLocation">
                  <v-icon>{{ mdiPencil }}</v-icon>
                </v-btn>
              </v-col>
              <v-col cols="6">
                <div>{{ $t('modals.detail.pageOverview.fileCount') }}:</div>
                <div>{{ selectedFileCount }} / {{ torrentFileCount }}</div>
                <div v-if="selectedFileCount === 1">{{ torrentFileName }}</div>
                <v-btn v-if="selectedFileCount === 1" fab x-small color="accent" @click="renameTorrentFile">
                  <v-icon>{{ mdiPencil }}</v-icon>
                </v-btn>
              </v-col>
            </v-row>
          </v-card-text>
        </v-card>
      </v-col>
      <v-col cols="12" md="6">
        <v-card flat>
          <v-card-text>
            <v-row>
              <v-col cols="6">
                <div>{{ $t('torrent.properties.status') }}:</div>
                <v-chip small :class="torrentStateClass" class="white--text caption">{{ torrent.state }}</v-chip>
              </v-col>
              <v-col cols="6">
                <div>{{ $t('torrent.properties.category') }}:</div>
                <v-chip small class="upload white--text caption">
                  {{ torrent.category.length ? torrent.category : $t('navbar.filters.uncategorized') }}
                </v-chip>
              </v-col>
            </v-row>
            <v-row>
              <v-col cols="6">
                <div>{{ $t('torrent.properties.tracker') }}:</div>
                <v-chip small class="moving white--text caption">{{ this.torrent?.tracker ? getDomainBody(this.torrent?.tracker) : $t('navbar.filters.untracked') }}</v-chip>
              </v-col>
              <v-col cols="6" class="d-flex flex-wrap chipgap">
                <div>{{ $t('torrent.properties.tags') }}:</div>
                <v-chip v-if="torrent?.tags" v-for="tag in torrent.tags" :key="tag" small class="tags white--text caption">
                  {{ tag }}
                </v-chip>
                <v-chip v-if="!torrent?.tags || torrent.tags.length === 0" small class="tags white--text caption">
                  {{ $t('navbar.filters.untagged') }}
                </v-chip>
              </v-col>
            </v-row>
            <v-row>
              <v-col cols="6">
                <div>{{ $t('modals.detail.pageOverview.selectedFileSize') }}:</div>
                <div>{{ torrent?.size | formatData(shouldUseBinaryData()) }} / {{ torrent?.total_size | formatData(shouldUseBinaryData()) }}</div>
              </v-col>
              <v-col cols="6">
                <div>{{ $t('ratio') }}:</div>
                <div>{{ torrent?.ratio }}</div>
              </v-col>
            </v-row>
            <v-row>
              <v-col cols="6">
                <div>{{ $t('downloaded') }}:</div>
                <div>{{ torrent?.downloaded | formatData(shouldUseBinaryData()) }}</div>
              </v-col>
              <v-col cols="6">
                <div>{{ $t('uploaded') }}:</div>
                <div>{{ torrent?.uploaded | formatData(shouldUseBinaryData()) }}</div>
              </v-col>
            </v-row>
            <v-row>
              <v-col cols="6">
                <div>{{ $t('modals.detail.pageOverview.dlSpeedAverage') }}:</div>
                <div>{{ downloadSpeedAvg | formatSpeed(shouldUseBitSpeed()) }}</div>
              </v-col>
              <v-col cols="6">
                <div>{{ $t('modals.detail.pageOverview.upSpeedAverage') }}:</div>
                <div>{{ uploadSpeedAvg | formatSpeed(shouldUseBitSpeed()) }}</div>
              </v-col>
            </v-row>
          </v-card-text>
        </v-card>
      </v-col>
    </v-row>
  </v-flex>
</template>

<script lang="ts">
import dayjs from 'dayjs'
import { FullScreenModal, General, TorrentDashboardItem } from '@/mixins'
import qbit from '@/services/qbit'
import { getDomainBody, splitByUrl, stringContainsUrl } from '@/helpers'
import { defineComponent } from 'vue'
import { mapState } from 'vuex'
import { mdiArrowDown, mdiArrowUp, mdiCheck, mdiClose, mdiPencil } from '@mdi/js'
import { TorrentState } from '@/enums/vuetorrent'
import { Priority } from '@/enums/qbit'
import { TorrentFile } from '@/types/qbit/models'
import { formatDataUnit, formatDataValue } from '@/filters'

export default defineComponent({
  name: 'Overflow',
  mixins: [General, FullScreenModal, TorrentDashboardItem],
  props: {
    isActive: Boolean
  },
  data() {
    return {
      commonStyle: 'caption',
      comment: '',
      createdBy: '',
      creationDate: '',
      downloadSpeedAvg: 0,
      files: [] as TorrentFile[],
      isPrivateTorrent: false,
      selectedFileCount: 0,
      torrentFileCount: 0,
      torrentFileName: '',
      torrentPieceSize: 0,
      torrentPieceOwned: 0,
      torrentPieceCount: 0,
      uploadSpeedAvg: 0,
      mdiArrowUp,
      mdiArrowDown,
      mdiCheck,
      mdiClose,
      mdiPencil,
      TorrentState
    }
  },
  async mounted() {
    await this.getTorrentProperties()
    await this.updateSelectedFiles()
    if (this.shouldRenderPieceStates) {
      await this.renderTorrentPieceStates()
    }
  },
  computed: {
    ...mapState(['webuiSettings']),
    pieceSize() {
      return `${parseInt(formatDataValue(this.torrentPieceSize, true))} ${formatDataUnit(this.torrentPieceSize, true)}`
    },
    isFetchingMetadata() {
      return this.torrent?.state === TorrentState.METADATA
    },
    shouldRenderPieceStates() {
      return !this.isFetchingMetadata && this.torrentPieceCount > 0 && this.torrentPieceCount < this.webuiSettings.torrentPieceCountRenderThreshold
    }
  },
  watch: {
    async torrent() {
      if (!this.isActive) return

      await this.getTorrentProperties()
      await this.updateSelectedFiles()
      if (this.shouldRenderPieceStates) {
        await this.renderTorrentPieceStates()
      }
    }
  },
  methods: {
    getDomainBody,
    async getTorrentProperties() {
      const props = await qbit.getTorrentProperties(this.torrent?.hash as string)
      this.comment = props.comment
      this.createdBy = props.created_by
      this.creationDate = dayjs(props.creation_date * 1000).format(this.webuiSettings.dateFormat)
      this.downloadSpeedAvg = props.dl_speed_avg
      this.isPrivateTorrent = props.is_private
      this.torrentPieceSize = props.piece_size
      this.torrentPieceOwned = props.pieces_have
      this.torrentPieceCount = props.pieces_num
      this.uploadSpeedAvg = props.up_speed_avg
    },
    async renderTorrentPieceStates() {
      const canvas: HTMLCanvasElement | null = document.querySelector('canvas#pieceStates')
      if (canvas === null) return

      const pieces = await qbit.getTorrentPieceStates(this.torrent?.hash as string)

      // Source: https://github.com/qbittorrent/qBittorrent/blob/6229b817300344759139d2fedbd59651065a561d/src/webui/www/private/scripts/prop-general.js#L230
      canvas.width = pieces.length || -1
      const ctx = canvas.getContext('2d') as CanvasRenderingContext2D
      ctx.clearRect(0, 0, canvas.width, canvas.height)

      // Group contiguous colors together and draw as a single rectangle
      let color = ''
      let rectWidth = 1

      for (let i = 0; i < pieces.length; ++i) {
        const status = pieces[i]
        let newColor = ''

        if (status === 1)
          // requested / downloading
          newColor = this.$vuetify.theme.currentTheme['torrent-downloading'] as string
        else if (status === 2)
          // already downloaded
          newColor = this.$vuetify.theme.currentTheme['torrent-done'] as string
        else {
          // pending download
          const selected_piece_ranges = this.files.filter(file => file.priority !== 0).map(file => file.piece_range)
          for (const [min_piece_range, max_piece_range] of selected_piece_ranges) {
            if (i > min_piece_range && i < max_piece_range) {
              newColor = this.$vuetify.theme.currentTheme['torrent-paused'] as string
              break
            }
          }
        }

        if (newColor === color) {
          ++rectWidth
          continue
        }

        if (color !== '') {
          ctx.fillStyle = color
          ctx.fillRect(i - rectWidth, 0, rectWidth, canvas.height)
        }

        rectWidth = 1
        color = newColor
      }

      // Fill a rect at the end of the canvas if one is needed
      if (color !== '') {
        ctx.fillStyle = color
        ctx.fillRect(pieces.length - rectWidth, 0, rectWidth, canvas.height)
      }
    },
    async updateSelectedFiles() {
      this.files = await qbit.getTorrentFiles(this.torrent?.hash as string)
      const selectedFiles = this.files.filter(f => f.priority != Priority.DO_NOT_DOWNLOAD)

      this.selectedFileCount = selectedFiles.length
      this.torrentFileCount = this.files.length
      if (this.selectedFileCount === 1) {
        this.torrentFileName = selectedFiles[0].name
      }
    },
    stringContainsUrl(string: string) {
      return stringContainsUrl(string)
    },
    splitString(string: string) {
      return splitByUrl(string)
    },
    async copyHash() {
      try {
        await navigator.clipboard.writeText(this.torrent?.hash as string)
        this.$toast.success(this.$t('toast.copySuccess').toString())
      } catch (err) {
        this.$toast.error(this.$t('toast.copyNotSupported').toString())
      }
    },
    async changeLocation() {
      this.createModal('ChangeLocationModal', { hashes: [this.torrent?.hash as string] })
    },
    async renameTorrentFile() {
      this.createModal('RenameTorrentFileModal', {
        hash: this.torrent?.hash as string,
        isFolder: false,
        oldName: this.torrentFileName
      })
    }
  }
})
</script>

<style lang="scss" scoped>
:deep(.v-data-table thead th),
:deep(.v-data-table tbody td) {
  padding: 0 !important;
  height: 3em;

  &:first-child {
    padding: 0 0 0 8px !important;
  }

  &:last-child {
    padding-right: 8px !important;
  }
}

:deep(.v-data-table tbody td.caption) {
  width: 20%;
}

canvas#pieceStates {
  height: 100%;
  width: 100%;
  border: 1px dotted;
}

.v-card__title {
  word-break: normal;
}

.chipgap {
  gap: 4px;
}

.overflow-wrap {
  overflow-wrap: anywhere;
}
</style>
