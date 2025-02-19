<template>
  <div sticky-container>
    <pull-to-reveal>
      <navbar view-name="prints"></navbar>
    </pull-to-reveal>

    <div
      class="menu-bar px-sm-4 d-flex justify-content-between align-items-center"
      v-sticky
      sticky-offset="{top: 0, bottom: 30}"
      sticky-side="both"
      on-stick="onMenuStick"
    >
      <b-form-checkbox
        v-model="allPrintsSelected"
        size="lg"
        class="text-decoration-none"
      ></b-form-checkbox>
      <div>
        <b-dropdown
          toggle-class="text-decoration-none no-corner no-border no-shadow"
          :variant="filterBtnVariant"
          no-caret
        >
          <template v-slot:button-content>
            <i class="fas fa-filter"></i>
          </template>
          <b-dropdown-item @click="onFilterClick('none')">
            <i class="fas fa-check" :style="{visibility: filter === 'none' ? 'visible' : 'hidden'}"></i>All
          </b-dropdown-item>
          <b-dropdown-divider></b-dropdown-divider>
          <b-dropdown-item @click="onFilterClick('finished')">
            <i
              class="fas fa-check"
              :style="{visibility: filter === 'finished' ? 'visible' : 'hidden'}"
            ></i>Finished
          </b-dropdown-item>
          <b-dropdown-item @click="onFilterClick('cancelled')">
            <i
              class="fas fa-check"
              :style="{visibility: filter === 'cancelled' ? 'visible' : 'hidden'}"
            ></i>Cancelled
          </b-dropdown-item>
          <b-dropdown-item @click="onFilterClick('need_alert_overwrite')">
            <i
              class="fas fa-check"
              :style="{visibility: filter === 'need_alert_overwrite' ? 'visible' : 'hidden'}"
            ></i>Review needed
          </b-dropdown-item>
          <b-dropdown-item @click="onFilterClick('need_print_shot_feedback')">
            <i
              class="fas fa-check"
              :style="{visibility: filter === 'need_print_shot_feedback' ? 'visible' : 'hidden'}"
            ></i>Focused-review needed
          </b-dropdown-item>
        </b-dropdown>
        <b-dropdown
          toggle-class="text-decoration-none no-corner no-border no-shadow"
          variant="outline-secondary"
          no-caret
        >
          <template v-slot:button-content>
            <i class="fas" :class="sortingBtnClasses"></i>
          </template>
          <b-dropdown-item @click="onSortingClick('date_desc')">
            <i
              class="fas fa-check"
              :style="{visibility: sorting === 'date_desc' ? 'visible' : 'hidden'}"
            ></i>Newest to oldest
          </b-dropdown-item>
          <b-dropdown-item @click="onSortingClick('date_asc')">
            <i
              class="fas fa-check"
              :style="{visibility: sorting === 'date_asc' ? 'visible' : 'hidden'}"
            ></i>Oldest to newest
          </b-dropdown-item>
        </b-dropdown>

        <button
          type="button"
          class="btn mx-2 btn-sm"
          :class="{'btn-light': !anyPrintsSelected, 'btn-danger': anyPrintsSelected}"
          :disabled="!anyPrintsSelected"
          @click="onDeleteBtnClick"
        >
          <i class="fas fa-trash-alt"></i>
          Delete {{ anyPrintsSelected ? ' (' + selectedPrintIds.size + ')' : '' }}
        </button>
        <a role="button" class="btn btn-sm btn-outline-primary upload-icon" href="/prints/upload/">
          <i class="fas fa-upload fa-lg mx-2"></i>
        </a>
      </div>
    </div>
    <div class="row">
      <print-card
        v-for="print of prints"
        :key="print.id"
        :print="print"
        :selected="selectedPrintIds.has(print.id)"
        @selectedChanged="onSelectedChanged"
        @printDeleted="onPrintDeleted"
        @printDataChanged="printDataChanged"
        @fullscreen="openFullScreen"
      ></print-card>
    </div>

    <mugen-scroll :handler="fetchMoreData" :should-handle="!loading" class="text-center p-4">
      <div v-if="noMoreData" class="text-center p-2">End of your time-lapse list.</div>
      <b-spinner v-if="!noMoreData" label="Loading..."></b-spinner>
    </mugen-scroll>

    <b-modal
      id="tl-fullscreen-modal"
      size="full"
      @hidden="fullScreenClosed"
      :hideHeader="true"
      :hideFooter="true"
    >
      <FullScreenPrintCard
        :print="fullScreenPrint"
        :videoUrl="fullScreenPrintVideoUrl"
        :autoplay="true"
      />
    </b-modal>
  </div>
</template>

<script>
import axios from 'axios'
import findIndex from 'lodash/findIndex'
import MugenScroll from 'vue-mugen-scroll'
import map from 'lodash/map'

import urls from '../lib/server_urls'
import { normalizedPrint } from '../lib/normalizers'
import PrintCard from './PrintCard.vue'
import FullScreenPrintCard from './FullScreenPrintCard.vue'
import PullToReveal from '@common/PullToReveal.vue'
import Navbar from '@common/Navbar.vue'

export default {
  name: 'PrintsPage',
  components: {
    MugenScroll,
    PrintCard,
    FullScreenPrintCard,
    PullToReveal,
    Navbar,
  },
  data: function() {
    return {
      prints: [],
      selectedPrintIds: new Set(),
      loading: false,
      noMoreData: false,
      filter: 'none',
      sorting: 'date_desc',
      fullScreenPrint: null,
      fullScreenPrintVideoUrl: null
    }
  },
  computed: {
    filterBtnVariant() {
      return this.filter === 'none' ? 'outline-secondary' : 'outline-primary'
    },

    sortingBtnClasses() {
      return this.sorting === 'date_asc'
        ? ' fa-sort-amount-up'
        : 'fa-sort-amount-down'
    },

    anyPrintsSelected() {
      return this.selectedPrintIds.size > 0
    },

    allPrintsSelected: {
      get: function () {
        return this.selectedPrintIds.size >= this.prints.length
      },
      set: function (selected) {
        if (selected) {
          this.selectedPrintIds = new Set(map(this.prints, 'id'))
        }
      }
    },
  },

  methods: {
    fetchMoreData() {
      if (this.noMoreData) {
        return
      }

      this.loading = true
      axios
        .get(urls.prints(), {
          params: {
            start: this.prints.length,
            limit: 6,
            filter: this.filter,
            sorting: this.sorting
          }
        })
        .then(response => {
          this.loading = false
          this.noMoreData = response.data.length < 6
          this.prints.push(...response.data.map(data => normalizedPrint(data)))
        })
    },

    refetchData() {
      this.prints = []
      this.selectedPrintIds = new Set()
      this.noMoreData = false
      this.fetchMoreData()
    },

    onSelectedChanged(printId, selected) {
      const selectedPrintIdsClone = new Set(this.selectedPrintIds)
      if (selected) {
        selectedPrintIdsClone.add(printId)
      } else {
        selectedPrintIdsClone.delete(printId)
      }
      this.selectedPrintIds = selectedPrintIdsClone
    },

    onMenuStick(data) {
      console.log(data)
    },

    onFilterClick(filter) {
      this.filter = filter
      this.refetchData()
    },

    onSortingClick(sorting) {
      this.sorting = sorting
      this.refetchData()
    },

    onDeleteBtnClick() {
      const selectedPrintIds = Array.from(this.selectedPrintIds)
      this.$swal({
        title: 'Are you sure?',
        text: `Delete ${this.selectedPrintIds.size} print(s)? This action can not be undone.`,
        showCancelButton: true,
        confirmButtonText: 'Yes',
        cancelButtonText: 'No'
      }).then(userAction => {
        if (userAction.isConfirmed) {
          axios
            .post(urls.printsBulkDelete(), { print_ids: selectedPrintIds })
            .then(() => {
              selectedPrintIds.forEach(printId => this.onPrintDeleted(printId, false))
              this.$swal.Toast.fire({
                title: `${selectedPrintIds.length} time-lapse(s) deleted!`,
              })
              this.selectedPrintIds = new Set()
            })
        }
      })
    },
    onPrintDeleted(printId, toast=true) {
      const i = findIndex(this.prints, p => p.id == printId)
      const print = this.prints[i]
      this.$delete(this.prints, i)
      if (toast) {
        this.$swal.Toast.fire({
          title: `Time-lapse ${print.filename} deleted!`,
        })
      }
    },
    printDataChanged(data) {
      const i = findIndex(this.prints, p => p.id == data.id)
      this.$set(this.prints, i, normalizedPrint(data))
    },
    openFullScreen(printId, videoUrl) {
      const i = findIndex(this.prints, p => p.id == printId)
      if (i != -1) {
        this.fullScreenPrint = this.prints[i]
        this.fullScreenPrintVideoUrl = videoUrl
        this.$bvModal.show('tl-fullscreen-modal')
      }
    },
    fullScreenClosed() {
      this.fullScreenPrint = null
      this.fullScreenPrintVideoUrl = null
    }
  }
}
</script>

<style lang="sass" scoped>
@use "~main/theme"

.menu-bar
  background-color: rgb(var(--color-bg-dark-d-10))
  padding: 0.75rem 1.25rem

#tl-fullscreen-modal
  .modal-full
    max-width: 100%

  .modal-body
    padding: 0

  .video-js
    height: calc(100vh - 200px)

.btn-outline-primary.upload-icon
  color: rgb(var(--color-white-primary))
  border-color: rgb(var(--color-white-primary))

  &:hover
    color: rgb(var(--color-primary-white))
    background-color: rgb(var(--color-white-primary))

::v-deep .btn-outline-secondary
  color: #fff
  border-color: #fff

  &:hover
    color: rgb(var(--color-text-inverted))
</style>
