<template>
  <div>
    <pull-to-reveal>
      <navbar view-name="printer_shared"></navbar>
    </pull-to-reveal>

    <div class="row justify-content-center">
      <b-spinner v-if="loading" class="mt-5" label="Loading..."></b-spinner>
      <div v-if="printer"
        class="col-sm-12 col-lg-6 printer-card"
      >
        <div class="card">
          <div class="card-header">
            <div>{{ printer.name }}</div>
          </div>
          <streaming-box :printer="printer" :webrtc="webrtc" />
          <div class="p-3 p-md-5">
            <p class="text-center">You are viewing an awesome 3D print your friend shared specifically with you on</p>
            <a
              href="https://www.thespaghettidetective.com/">
              <dark-light-image path="navbar-brand" ext="png" alt="The Spaghetti Detective" class="logo-img"></dark-light-image>
            </a>
            <hr />
            <br /><br />
            <a class="btn btn-block btn-primary" href="/accounts/signup/">Get OctoPrint remote monitoring/access for FREE</a>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import split from 'lodash/split'

import { normalizedPrinter } from '@lib/normalizers'
import urls from '@lib/server_urls'
import PrinterComm from '@lib/printer_comm'
import WebRTCConnection from '@lib/webrtc'
import StreamingBox from '@common/StreamingBox'
import PullToReveal from '@common/PullToReveal.vue'
import Navbar from '@common/Navbar.vue'
import DarkLightImage from '@common/DarkLightImage.vue'

export default {
  name: 'SharedPrinterPage',
  components: {
    StreamingBox,
    PullToReveal,
    Navbar,
    DarkLightImage,
  },
  created(){
    this.shareToken = split(window.location.pathname, '/').slice(-2, -1).pop()

    this.printerComm = PrinterComm(
      this.shareToken,
      urls.printerSharedWebSocket(this.shareToken),
      (data) => {
        this.printer = normalizedPrinter(data, this.printer)
        this.loading = false
      }
    )
    this.printerComm.connect()
    this.webrtc.openForShareToken(this.shareToken)
  },
  data: function() {
    return {
      printer: null,
      shareToken: null,
      videoAvailable: {},
      loading: true,
      webrtc: WebRTCConnection(true),
    }
  },
}
</script>

<style lang="sass" scoped>
@use "~main/theme"

#printer-list-page
  margin-top: 1.5rem

.menu-bar
  background-color: rgb(var(--color-bg-dark-d-10))
  padding: 0.75rem

.printer-card
  margin-bottom: 1.5rem

::v-deep .logo-img img
  width: 100%
</style>
