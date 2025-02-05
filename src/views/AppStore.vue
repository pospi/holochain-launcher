<template>
  <HCLoading ref="downloading" :text="loadingText" />

  <div v-if="loading" class="column center-content" style="flex: 1; min-height: calc(100vh - 64px);">
    <LoadingDots style="--radius: 15px; --dim-color: #e8e8eb; --fill-color: #b5b5b5;"></LoadingDots>
  </div>

  <div
    v-else-if="installableApps.length === 0"
    class="column center-content"
    style="flex: 1; min-height: calc(100vh - 64px);"
  >
    <span>{{ $t("appStore.noAppsInStore") }}</span>
    <HCButton
      outlined
      @click="fetchApps()"
      class="refresh-button"
      >{{ $t("main.refresh") }}
    </HCButton>
  </div>

  <div v-else class="row" style="flex-wrap: wrap; margin: 16px; min-height: calc(100vh - 64px); margin-bottom: 200px; align-content: flex-start;">
    <div
      v-for="(app, i) of installableApps"
      :key="i"
      class="column"
      style="margin-right: 16px; margin-bottom: 16px"
    >
      <AppPreviewCard :app="app" :appWebsocket="appWebsocket" @installApp="requestInstall(app, $event.imgSrc)" />
    </div>
  </div>

  <!-- AppStore synchronization spinner -->
  <div v-show="showLoadingSpinner" class="progress-indicator">
    <div style="padding: 0 15px;">
      <div
        style="margin-bottom: 5px; font-weight: 600; font-size: 18px;"
        :title="$t('appStore.fullSynchronizationRequired')"
      >
        {{ $t('appStore.receivingData') }}...
      </div>
      <div style="text-align: right; margin-bottom: 10px;" :title="$t('appStore.amountOfData')">
        <b>{{ prettyBytesLocal(queuedBytes) }}</b> {{ $t('appStore.inQueue') }}
      </div>
    </div>
    <span :class="queuedBytes ? 'loader' : 'inactive-loader'" style="position: absolute; bottom: 0;"></span>
  </div>

  <!-- Select from filesystem button -->
   <HCButton
    style="
      height: 40px;
      border-radius: 8px;
      padding: 0 20px;
      position:fixed;
      bottom: 20px;
      left: 50%;
      margin-left: -140px;
    "
    @click="selectFromFileSystem()"
  >
    <div class="row center-content">
      <mwc-icon>folder</mwc-icon>
      <span style="margin-left: 5px">{{
        $t("appStore.selectAppFromFileSystem")
      }}</span>
    </div>
  </HCButton>


  <!-- Dialog to select releases -->
  <SelectReleaseDialog
    v-if="selectedApp"
    :app="selectedApp"
    :appWebsocket="appWebsocket"
    :imgSrc="selectedIconSrc"
    ref="selectAppReleasesDialog"
    @cancel="() => {
      selectedApp = undefined;
    }"
    @release-selected="saveApp($event)"
  >
  </SelectReleaseDialog>

  <InstallAppDialog
    v-if="selectedAppBundlePath"
    :appBundlePath="selectedAppBundlePath"
    :holochainSelection="holochainSelection"
    :happReleaseInfo="selectedHappReleaseInfo"
    :guiReleaseInfo="selectedGuiReleaseInfo"
    :iconSrc="selectedIconSrc"
    @app-installed="
      holochainSelection = true;
      installClosed();
      showMessage(`Installed App ${$event}`);
      $emit('select-view', { type: 'launcher' });;
    "
    @closing-dialog="installClosed()"
    @error="(e) => showMessage(e)"
    ref="install-app-dialog"
  ></InstallAppDialog>
  <HCSnackbar
    :labelText="errorText"
    ref="snackbar"
  ></HCSnackbar>
</template>

<script lang="ts">
import { defineComponent } from "vue";
import "@material/mwc-circular-progress";
import "@material/mwc-icon";
import "@material/mwc-icon-button";
import { AppWebsocket, NetworkInfo, CellInfo, encodeHashToBase64 } from "@holochain/client";
import { open } from "@tauri-apps/api/dialog";
import { invoke } from "@tauri-apps/api/tauri";
import { toSrc, getCellId } from "../utils";

import HCSnackbar from "../components/subcomponents/HCSnackbar.vue";
import HCProgressBar from "../components/subcomponents/HCProgressBar.vue";
import LoadingDots from "../components/subcomponents/LoadingDots.vue";

import { tryWithHosts } from "../appstore/appstore-interface";
import InstallAppDialog from "../components/InstallAppDialog.vue";
import HCButton from "../components/subcomponents/HCButton.vue";
import AppPreviewCard from "../components/AppPreviewCard.vue";
import HCLoading from "../components/subcomponents/HCLoading.vue";
import HCDialog from "../components/subcomponents/HCDialog.vue";
import SelectReleaseDialog from "../components/SelectReleaseDialog.vue";

import { HolochainId, ReleaseData, ReleaseInfo } from "../types";
import prettyBytes from "pretty-bytes";
import { AppEntry } from "../appstore/types";
import { getAllApps } from "../appstore/appstore-interface";
import { APPSTORE_APP_ID } from "../constants";



export default defineComponent({
  name: "AppStore",
  components: {
    InstallAppDialog,
    HCButton,
    AppPreviewCard,
    HCLoading,
    HCSnackbar,
    HCProgressBar,
    HCDialog,
    LoadingDots,
    SelectReleaseDialog,
  },
  data(): {
    appWebsocket: AppWebsocket | undefined;
    loadingText: string;
    loading: boolean;
    installableApps: Array<AppEntry>;
    selectedAppBundlePath: string | undefined;
    holochainId: HolochainId | undefined;
    holochainSelection: boolean;
    provisionedCells: [string, CellInfo | undefined][] | undefined;
    networkStates: (number | undefined)[];
    cachedMaxExpected: (number | undefined)[];
    latestQueuedBytesUpdate: number;
    showProgressIndicator: boolean;
    errorText: string;
    pollInterval: number | null;
    queuedBytes: number | undefined;
    selectedHappReleaseInfo: ReleaseInfo | undefined;
    selectedGuiReleaseInfo: ReleaseInfo | undefined;
    selectedApp: AppEntry | undefined;
    selectedIconSrc: string | undefined;
    showLoadingSpinner: boolean;
  } {
    return {
      appWebsocket: undefined,
      loadingText: "",
      loading: true,
      installableApps: [],
      selectedAppBundlePath: undefined,
      holochainId: undefined,
      holochainSelection: true,
      provisionedCells: undefined,
      networkStates: [undefined, undefined, undefined],
      cachedMaxExpected: [undefined, undefined, undefined],
      latestQueuedBytesUpdate: 0,
      showProgressIndicator: false,
      errorText: "Unknown error occured.",
      pollInterval: null,
      queuedBytes: undefined,
      selectedHappReleaseInfo: undefined,
      selectedGuiReleaseInfo: undefined,
      selectedApp: undefined,
      selectedIconSrc: undefined,
      showLoadingSpinner: false,
    };
  },
  beforeUnmount() {
    window.clearInterval(this.pollInterval!);
  },
  async mounted() {
    try {
      await this.fetchApps();
    } catch (e) {
      console.error(`Failed to fetch apps in mounted() hook: ${e}`);
    }

    await this.getQueuedBytes();
    this.pollInterval = window.setInterval(
      async () => await this.getQueuedBytes(),
      2000
    );

    // If the "Filesystem" button is pressed in the "launcher" view with no apps installed, the
    // "installFromFs" item is set to "true" in localStorage and then the view is switched to
    // "appStore" view (i.e. to this component here).
    // In that case, the select from filesystem logic shall immediately be called after mounting of the component
    // and the localStorage item be removed again.
    if (window.localStorage.getItem("installFromFs")) {
      window.localStorage.removeItem("installFromFs");
      this.selectFromFileSystem();
    }
  },
  methods: {
    toSrc,
    async connectAppWebsocket() {
      // const _hdiOfDevhub = this.$store.getters["hdiOfDevhub"]; // currently not used
      const holochainId = this.$store.getters["holochainIdForDevhub"];
      this.holochainId = holochainId;
      // connect to AppWebsocket
      const port = this.$store.getters["appInterfacePort"](holochainId);
      this.appWebsocket = await AppWebsocket.connect(`ws://localhost:${port}`, 40000);
      // console.log("connected to AppWebsocket.");
    },
    async fetchApps() {

      this.loading = true;

      if (!this.appWebsocket) {
        await this.connectAppWebsocket();
      }

      const appStoreInfo = await this.appWebsocket!.appInfo({
        installed_app_id: APPSTORE_APP_ID,
      });

      console.log("@fetchApps: appStoreInfo: ", appStoreInfo);
      const allCells = appStoreInfo.cell_info;
      console.log("@fetchApps: allCells: ", allCells);

      const provisionedCells: [string, CellInfo | undefined][] = Object.entries(allCells).map(([roleName, cellInfos]) => {
        return [roleName, cellInfos.find((cellInfo) => "provisioned" in cellInfo)]
      });

      console.log("@fetchApps: provisionedCells: ", provisionedCells);

      this.provisionedCells = provisionedCells.sort(([roleName_a, _cellInfo_a], [roleName_b, _cellInfo_b]) => {
        return roleName_a.localeCompare(roleName_b);
      });


      let allApps: Array<AppEntry>;
      try {
        allApps = await getAllApps((this.appWebsocket! as AppWebsocket), appStoreInfo);
      } catch (e) {
        console.error(`Error getting all apps: ${e}`);
        // Catch other errors than being offline
        allApps = [];
      }

      console.log("@fetchApps: allApps: ", allApps);

      // filter by apps of the relevant DevHub dna hash
      // this.installableApps = allApps.filter((appEntry) => JSON.stringify(appEntry.devhub_address.dna) === JSON.stringify(DEVHUB_HAPP_LIBRARY_DNA_HASH));

      this.installableApps = allApps;

      this.loading = false;

      // console.log("ALL APPS: ", allApps);
      // console.log("FILTERED APPS: ", this.installableApps);
      // console.log("hdk versions: ", hdk_versions);
    },
    async peerToPeer() {
      await invoke("open_url_cmd", {
        url: "https://developer.holochain.org/glossary/#peer-to-peer",
      });
    },
    /**
     *
     */
    async requestInstall(app: AppEntry, imgSrc: string | undefined) {

      this.selectedIconSrc = imgSrc ? imgSrc : undefined;
      this.selectedApp = app;

      // 1. get happ releases for app from DevHub
      if (!this.appWebsocket) {
        await this.connectAppWebsocket();
      }

      this.$nextTick(() => {
        (this.$refs.selectAppReleasesDialog as typeof SelectReleaseDialog).open();
      });
    },
    async saveApp(releaseInfo: ReleaseData) {
      // // if downloading, always take holochain version of DevHub
      this.holochainSelection = false;
      this.loadingText = "searching available peer host";
      (this.$refs.downloading as typeof HCLoading).open();


      const appStoreInfo = await this.appWebsocket!.appInfo({
        installed_app_id: APPSTORE_APP_ID,
      });

      const happReleaseHash = releaseInfo.happRelease.id;
      const guiReleaseHash = releaseInfo.happRelease.content.official_gui;

      this.selectedHappReleaseInfo = {
        resource_locator: {
          dna_hash: encodeHashToBase64(releaseInfo.devhubDnaHash),
          resource_hash: encodeHashToBase64(happReleaseHash),
        },
        version: releaseInfo.happRelease.content.version,
      };
      this.selectedGuiReleaseInfo = guiReleaseHash ? {
        resource_locator: {
          dna_hash: encodeHashToBase64(releaseInfo.devhubDnaHash),
          resource_hash: encodeHashToBase64(guiReleaseHash),
        },
        version: releaseInfo.guiRelease?.content.version
      } : undefined;

      this.loadingText = `fetching app from peer host...`;

      try {
        await tryWithHosts<void>(
          async (host) => {
            this.selectedAppBundlePath = await invoke("fetch_and_save_app", {
              holochainId: this.holochainId,
              appstoreAppId: appStoreInfo.installed_app_id,
              appTitle: this.selectedApp!.title,
              host: Array.from(host),
              devhubHappLibraryDnaHash: Array.from(releaseInfo.devhubDnaHash), // DNA hash of the DevHub to which the remote call shall be made
              appstorePubKey: encodeHashToBase64(appStoreInfo.agent_pub_key),
              happReleaseHash: encodeHashToBase64(happReleaseHash),
              guiReleaseHash: guiReleaseHash ? encodeHashToBase64(guiReleaseHash) : undefined,
            });

            (this.$refs.downloading as typeof HCLoading).close();
            this.loadingText = "";

            this.$nextTick(() => {
              (this.$refs["install-app-dialog"] as typeof InstallAppDialog).open();
            });

            console.log("@saveApp: selectedAppBundlePath: ", this.selectedAppBundlePath);

          },
          this.appWebsocket as AppWebsocket,
          appStoreInfo,
          releaseInfo.devhubDnaHash,
          "happ_library",
          "get_webhapp_package",
        )

      } catch (e) {
        console.error("Error fetching webhapp from DevHub host(s): ", e);
        this.selectedHappReleaseInfo = undefined;
        this.selectedGuiReleaseInfo = undefined;
        this.selectedApp = undefined;
        this.selectedIconSrc = undefined;
        this.showMessage("Failed to fetch webhapp from DevHub host(s).");
        (this.$refs.downloading as typeof HCLoading).close();
        return;
      }
    },
    async selectFromFileSystem() {
      this.selectedAppBundlePath = (await open({
        filters: [
          { name: "Holochain Application", extensions: ["webhapp", "happ"] },
        ],
      })) as string;

      this.$nextTick(() => {
        (this.$refs["install-app-dialog"] as typeof InstallAppDialog).open();
      });
    },
    installClosed() {
      this.selectedAppBundlePath = undefined;
      this.selectedApp = undefined;
      // this.hdkVersionForApp = undefined;
    },
    /**
    * Gets aggregated bytes that are in queue for the DevHub cells
    */
    async getQueuedBytes() {
      if (!this.appWebsocket) {
        await this.connectAppWebsocket();
      }
      const networkInfo: NetworkInfo[] = await this.appWebsocket!.networkInfo({
        agent_pub_key: getCellId(this.provisionedCells![0][1]!)![1],
        dnas: this.provisionedCells!.filter(([_roleName, cellInfo]) => !!cellInfo)
          .map(([_roleName, cellInfo]) => getCellId(cellInfo!)![0] as Uint8Array),
      } as any);
      let queuedBytes = 0;
      networkInfo.forEach((info, _idx) => {
        queuedBytes += info.fetch_pool_info.op_bytes_to_fetch;
      });
      this.queuedBytes = queuedBytes;
      const now = Date.now();
      if (!!queuedBytes && queuedBytes > 0) {
        this.latestQueuedBytesUpdate = now;
        // console.log("updated timestamp: ", this.latestQueuedBytesUpdate);
      }
      if ((now - this.latestQueuedBytesUpdate) < 5000) {
        this.showLoadingSpinner = true;
      } else {
        this.showLoadingSpinner = false;
      }

      return queuedBytes;
    },
    showMessage(message: string) {
      this.$emit("show-message", message);
    },
    progressRatio(idx: number) {
      if ((this.networkStates[idx] || this.networkStates[idx] === 0) && this.cachedMaxExpected[idx]) {
        return (
          (1 - this.networkStates[idx]! / this.cachedMaxExpected[idx]!) * 100
        );
      } else {
        return undefined;
      }
    },
    prettyBytesLocal(input: number | undefined) {
      if (input || input === 0) {
        return prettyBytes(input);
      } else {
        return "-";
      }
    },
    byteDiff(idx: number) {
      const cachedMax = this.cachedMaxExpected[idx] ? this.cachedMaxExpected[idx] : 0;
      const currentExpected = this.networkStates[idx] ? this.networkStates[idx] : 0;
      const diff = cachedMax! - currentExpected!;

      if (diff < 0) {
        return 0;
      } else {
        return diff;
      }
    },
  },
});
</script>

<style scoped>
.top-bar {
  align-items: center;
  height: 64px;
  background: white;
  box-shadow: 0 0px 5px #9b9b9b;
}

.progress-indicator {
  position: fixed;
  bottom: 20px;
  right: 20px;
  padding: 10px 0 0 0;
  background-color: white;
  box-shadow: 0 0px 5px #9b9b9b;
  border-radius: 10px 10px 6px 6px;
  min-width: 350px;
}

.refresh-button {
  height: 30px;
  border-radius: 8px;
  padding: 0 15px;
  --hc-primary-color: #2c3e50;
  opacity: 0.75;
  margin-top: 30px;
}

.refresh-button:hover {
  opacity: 1;
}

</style>
