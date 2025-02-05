<template>

  <HCGenericDialog
    @confirm="updateGui"
    ref="updateGuiDialog"
    :primaryButtonLabel="$t('buttons.install')"
    :closeOnSideClick="true"
  >
    <div class="column" style="padding: 0 30px; align-items: flex-start; max-width: 500px;">
      <div style="width: 100%; text-align: center; font-weight: 600; font-size: 27px; margin-bottom: 25px">
        {{ $t("dialogs.guiUpdate.title") }}
      </div>
      <div style="margin-bottom: 15px;">
        {{ $t("dialogs.guiUpdate.mainText") }}:
      </div>
      <div>
        <span style="font-weight: bold; margin-right: 15px;">{{ $t("dialogs.guiUpdate.version") }}:</span>{{ selectedGuiUpdate ? selectedGuiUpdate.version : "loading..." }}
      </div>
      <div style="font-weight: bold;">
        {{ $t("dialogs.guiUpdate.changelog") }}:
      </div>
      <div style="background: rgb(217,217,217); border-radius: 8px; padding: 10px; width: 480px; min-height: 100px; max-height: 200px; overflow-y: auto; margin-top: 5px; white-space: pre-wrap;">
        {{ selectedGuiUpdate ? selectedGuiUpdate.changelog : "loading..." }}
      </div>
      <div style="margin-top: 20px;">
        {{ $t("dialogs.guiUpdate.question") }}
      </div>
    </div>
  </HCGenericDialog>

  <Config ref="configDialog"/>

  <HCLoading ref="downloading" :text="loadingText"></HCLoading>

  <div
    v-if="isLoading()"
    class="column center-content" style="flex: 1; height: calc(100vh - 64px);"
  >
    <LoadingDots style="--radius: 15px; --dim-color: #e8e8eb; --fill-color: #b5b5b5"></LoadingDots>
  </div>

  <div v-else style="display: flex; margin: 24px; margin-bottom: 50px; flex-direction: column; align-items: center;">
    <div class='column' style="flex: 1 1 0%; margin-bottom: 80px; padding: 0px 30px; width: 70%; min-width: 900px;">

      <!-- Holochain version info -->
      <div
        class="row section"
      >
        <span
          class="section-title"
          :title="$t('settings.holochainVersionsHelper')"
          >{{ $t("settings.holochainVersions") }}</span
        >
        <span style="flex: 1"></span>
        <span
          @click="refreshStorageInfo"
          style="margin-right: 5px; margin-bottom: -8px; cursor: pointer"
        >
          <img
            src="/img/refresh.png"
            style="height: 12px; margin-right: 3px; opacity: 0.7"
          />
          {{ $t("main.refresh") }}
        </span>
      </div>

      <div
        class="column"
        style="margin-bottom: 15px; width: 100%;"
      >
        <div
          v-if="noHolochainVersions"
          style="margin-top: 30px; color: rgba(0, 0, 0, 0.6); text-align: center"
        >
          {{ $t("settings.noHolochainVersions") }}
        </div>
        <div v-else>
          <div
            v-for="hcVersion in holochainVersions"
            :key="hcVersion"
            style="
              display: flex;
              flex: 1;
              flex-direction: column;
              width: 100%;
            "
          >
            <div class="row section-container hc-version" style="margin: 5px 0">
              <img
                src="/img/Square284x284Logo.png"
                style="height: 42px; margin-left: 11px; margin-right: 11px"
              />
              <div>
                <h2>{{ hcVersion }}</h2>
              </div>
              <span style="display: flex; flex: 1"></span>
              <span
                v-if="storageInfos && !refreshing"
                style="font-weight: 600; margin-right: 15px"
                >{{ totalStorageString(hcVersion) }}</span
              >
              <StackedChart
                v-if="storageInfos && !refreshing"
                :fractions="storageFractions(hcVersion)"
                :labels="storageLabels(hcVersion)"
                style="width: 200px; height: 34px; margin-right: 12px"
              ></StackedChart>
              <span style="width: 120px; text-align: center">{{
                storageInfos[hcVersion]
                  ? prettyBytes(storageInfos[hcVersion].conductor)
                  : "?"
              }}</span>
              <span style="width: 120px; text-align: center">{{
                storageInfos[hcVersion]
                  ? prettyBytes(storageInfos[hcVersion].uis)
                  : "?"
              }}</span>
            </div>
          </div>
        </div>
      </div>

      <!-- <div class="row section-container" style="display: flex; flex-direction: row;">
        Language
      </div> -->

      <!-- Advanced Settings Section -->
      <div
        class="row section"
        :class="{ borderBottomed: showAdvancedSettings }"
      >
        <span class="section-title">
          {{ $t("settings.advancedSettings") }}
        </span>
        <span
          @click="showAdvancedSettings = !showAdvancedSettings"
          class="show-hide"
          style="opacity: 0.7; cursor: pointer; margin-left: 10px"
        >
          {{ showAdvancedSettings ? "[-]" : "[show]" }}
        </span>
      </div>

      <div v-if="showAdvancedSettings" style="margin-bottom: 15px;">

        <!-- Dev Mode -->
        <div class="row section-container" style="display: flex; flex-direction: column;">
          <div class="row">
            <div style="flex: 1;">
              <span style="font-size: 18px">{{ $t("main.activateDevMode") }}</span>
            </div>
            <!-- Disable/enable switch -->
            <sl-tooltip
              class="tooltip"
              hoist
              placement="top"
              :content="devModeOn ? 'Disable Dev Mode' : 'Enable Dev Mode'"
            >
              <ToggleSwitch
                style="margin-right: 29px"
                :sliderOn="!!devHubAppInfo && isAppRunning(devHubAppInfo?.webAppInfo.installed_app_info)"
                @click.stop.prevent="toggleDevMode()"
                @keydown.enter="toggleDevMode()"
              />
            </sl-tooltip>
          </div>

          <div class="row">
            <HCButton
              outlined
              :disabled="!devModeOn"
              @click="openPublishAppDialog"
              style="height: 36px; border-radius: 8px; padding: 0 20px; margin-top: 10px;"
              >{{ $t("settings.publishAnApp") }}
            </HCButton>
          </div>
        </div>

        <div class="row section-container">
          <div class="column" style="flex: 1;">
            <div style="font-size: 18px">{{ $t("settings.launcherConfiguration") }}</div>
            <span style="font-size: 14px">{{ $t("settings.launcherConfigurationDescription") }}</span>
          </div>
          <HCButton style="margin: 4px 6px;" @click="openConfig()">Configure Launcher</HCButton>
        </div>
      </div>


      <!-- Installed apps list -->

      <div
        class="column"
        style="flex: 1; margin-top: 20px"
      >
        <div
          class="row"
          style="
            width: 100%;
            justify-content: flex-end;
            align-items: center;
            margin-bottom: -20px;
          "
        >
          <HCSelectCard
            style="
              width: 200px;
              margin-right: 5px;
              box-shadow: 0 0px 3px -1px #9b9b9b;
              --hc-label-background: #e8e8eb;
            "
            :placeholder="$t('settings.holochainVersions')"
            :items="holochainVersionOptions"
            @item-selected="selectedHolochainVersion = $event"
          ></HCSelectCard>
          <img
            src="/img/Square284x284Logo.png"
            style="
              height: 30px;
              filter: grayscale(50%);
              margin-right: 20px;
              margin-left: -2px;
            "
          />

          <HCSelectCard
            style="
              width: 200px;
              margin-right: 5px;
              box-shadow: 0 0px 3px -1px #9b9b9b;
              --hc-label-background: #e8e8eb;
            "
            :placeholder="$t('main.sortBy')"
            :items="sortOptions"
            @item-selected="sortOption = $event"
          ></HCSelectCard>
          <mwc-icon style="color: #482edf; text-shadow: 0 0px 5px #9b9b9b"
            >sort</mwc-icon
          >
        </div>

        <!-- Web Apps -->

        <div
          class="row section"
          :class="{ borderBottomed: showWebApps }"
          style="margin-top: -25px"
        >
          <span
            class="section-title"
            :title="$t('settings.appSettingsHelper')"
            >{{ $t("settings.appSettings") }}</span
          >
          <span
            @click="showWebApps = !showWebApps"
            class="show-hide"
            style="opacity: 0.7; cursor: pointer; margin-left: 10px"
          >
            {{ showWebApps ? "[-]" : "[show]" }}
          </span>
        </div>

        <div v-if="showWebApps" style="margin-bottom: 50px; padding: 0 15px;">
          <div
            v-if="noWebApps"
            style="margin-top: 30px; color: rgba(0, 0, 0, 0.6); text-align: center"
          >
            {{ $t("settings.noWebApps") }}
            {{
              selectedHolochainVersion === "All Versions"
                ? "."
                : " in this Holochain Version."
            }}
          </div>

          <div
            v-else
            v-for="app in sortedApps"
            :key="app.webAppInfo.installed_app_info.installed_app_id"
            style="
              display: flex;
              flex-direction: column;
              width: 100%;
              align-items: center;
            "
          >
            <AppSettingsCard
              v-if="app.webAppInfo.web_uis.default.type !== 'Headless'"
              :app="app"
              @openApp="openApp($event)"
              @uninstallApp="uninstallApp($event)"
              @disableApp="disableApp($event)"
              @enableApp="enableApp($event)"
              @startApp="startApp($event)"
              @updateGui="openUpdateGuiDialog($event)"
            />
          </div>
        </div>

        <!-- Headless Apps -->

        <div
          v-if="!noHeadlessApps"
          class="row section"
          :class="{ borderBottomed: showHeadlessApps }"
        >
          <span
            class="section-title"
            :title="$t('settings.headlessAppsHelper')"
            >{{ $t("settings.headlessApps") }}</span
          >
          <span
            @click="showHeadlessApps = !showHeadlessApps"
            class="show-hide"
            style="opacity: 0.7; cursor: pointer; margin-left: 10px"
          >
            {{ showHeadlessApps ? "[-]" : "[show]" }}
          </span>
        </div>

        <div v-if="showHeadlessApps && !noHeadlessApps" style="margin-bottom: 50px; padding: 0 15px;">
          <div
            v-for="app in sortedApps"
            :key="app.webAppInfo.installed_app_info.installed_app_id"
            style="
              display: flex;
              flex-direction: column;
              width: 100%;
              align-items: center;
            "
          >
            <AppSettingsCard
              v-if="app.webAppInfo.web_uis.default.type === 'Headless'"
              :app="app"
              @openApp="openApp($event)"
              @uninstallApp="uninstallApp($event)"
              @disableApp="disableApp($event)"
              @enableApp="enableApp($event)"
            />
          </div>
        </div>
      </div>
    </div>
  </div>

  <!-- Dialogs -->
  <HCDialog ref="devModeDevsOnlyWarning">
    <div
      class="column"
      style="padding: 30px; align-items: center; max-width: 500px"
    >
      <div style="font-weight: 600; font-size: 27px; margin-bottom: 25px">
        Dev Mode
      </div>
      <div>
        Turning on Dev Mode installs the DevHub app. DevHub is the place where
        <span style="font-weight: bold; white-space: nowrap;">app developers</span>
        can upload and manage their apps, and is required to publish apps to the AppStore.<br><br>
        Installing DevHub will download a lot of data, and synchronization with other DevHub nodes may take a long time.
        Are you sure you want to continue?
      </div>

      <div class="row" style="margin-top: 30px; margin-bottom: 10px; margin-left: 50px; width: 100%;">
        <ToggleSwitch
          :sliderOn="ignoreDevModeWarning"
          @click="() => ignoreDevModeWarning = !ignoreDevModeWarning"
          @keydown.enter="() => ignoreDevModeWarning = !ignoreDevModeWarning"
        />
        <span style="margin-left: 10px;">Don't show this message again.</span>
      </div>

      <div class="row" style="margin-top: 20px;">
        <HCButton style="height: 30px; margin: 4px 6px;" outlined @click="closeDevModeWarning">Cancel</HCButton>
        <HCButton style="margin: 4px 6px;" @click="handleInstallDevHub">Install DevHub</HCButton>
      </div>
    </div>
  </HCDialog>

  <HCDialog ref="publishAppDialog" close-on-side-click>
    <div
      class="column"
      style="padding: 30px; align-items: center; max-width: 500px"
    >
      <div style="font-weight: 600; font-size: 27px; margin-bottom: 25px">
        How to Publish An App
      </div>

      <div>
        To publish your own Holochain App you will need to upload it first to the Dev Hub and then to the App Store.
        First read the <a :href='howToPublishUrl' target="_blank">full instructions here</a>, then open the Dev Hub and App Store below.
      </div>

      <div class="row" style="margin-top: 20px;">
        <HCButton
          style="height: 30px; margin: 4px 6px;"
          outlined
          @click="devHubAppInfo ? openApp(devHubAppInfo) : undefined; "
        >
          {{ $t("settings.openDevHub") }}
        </HCButton>
        <HCButton
          style="margin: 4px 6px;"
          @click="appstoreHolochainAppInfo ? openApp(appstoreHolochainAppInfo) : undefined; closePublishAppDialog();"
        >
          {{ $t("settings.openAppStore") }}
        </HCButton>
      </div>
    </div>
  </HCDialog>

</template>

<script lang="ts">
import { AppInfo, AppWebsocket, decodeHashFromBase64, encodeHashToBase64, EntryHash, InstalledAppId, DnaHashB64 } from "@holochain/client";
import { uniq } from "lodash-es";
import prettyBytes from "pretty-bytes";
import { invoke } from "@tauri-apps/api/tauri";
import { defineComponent, PropType } from "vue";

import "@material/mwc-button";
import "@material/mwc-icon-button";
import "@material/mwc-icon";

import { getHappReleasesByEntryHashes, fetchGui, appstoreCells, fetchGuiReleaseEntry, tryWithHosts } from "../appstore/appstore-interface";
import { GUIReleaseEntry, HappReleaseEntry } from "../appstore/types";
import { APPSTORE_APP_ID, DEVHUB_APP_ID } from "../constants";
import AppSettingsCard from "../components/AppSettingsCard.vue";
import Config from "../components/settings/Config.vue";
import HCButton from "../components/subcomponents/HCButton.vue";
import HCDialog from "../components/subcomponents/HCDialog.vue";
import HCGenericDialog from "../components/subcomponents/HCGenericDialog.vue";
import HCLoading from "../components/subcomponents/HCLoading.vue";
import HCSelectCard from "../components/subcomponents/HCSelectCard.vue";
import HCSnackbar from "../components/subcomponents/HCSnackbar.vue";
import LoadingDots from "../components/subcomponents/LoadingDots.vue";
import ToggleSwitch from "../components/subcomponents/ToggleSwitch.vue";
import StackedChart from "../components/subcomponents/StackedChart.vue";
import { i18n } from "../locale";
import { ActionTypes } from "../store/actions";
import { HolochainAppInfo, HolochainAppInfoExtended, StorageInfo, ResourceLocator } from "../types";
import { isAppDisabled, isAppPaused, isAppRunning, locatorToLocatorB64 } from "../utils";

export default defineComponent({
  name: "Settings",
  components: {
    Config,
    HCButton,
    HCSnackbar,
    HCDialog,
    ToggleSwitch,
    LoadingDots,
    AppSettingsCard,
    HCSelectCard,
    StackedChart,
    HCGenericDialog,
    HCLoading
  },
  props: {
    installedApps: {
      type: Object as PropType<Array<HolochainAppInfo>>,
      required: true,
    },
  },
  data(): {
    appWebsocket: AppWebsocket | undefined;
    appstoreAppInfo: AppInfo | undefined;
    appstoreHolochainAppInfo: HolochainAppInfo | undefined;
    devHubAppInfo: HolochainAppInfo | null;
    devModeEnabled: boolean;
    errorText: string;
    extendedAppInfos: Record<InstalledAppId, HolochainAppInfoExtended> | undefined;
    howToPublishUrl: string;
    ignoreDevModeWarning: boolean;
    loadingText: string;
    refreshing: boolean;
    refreshTimeout: number | null;
    reportIssueUrl: string;
    selectedApp: HolochainAppInfoExtended | undefined;
    selectedGuiUpdate: GUIReleaseEntry | undefined;
    selectedGuiUpdateLocator: ResourceLocator | undefined;
    selectedHolochainVersion: string;
    showAdvancedSettings: boolean;
    showDevModeDevsOnlyWarning: boolean; // TODO: unused right now
    showHeadlessApps: boolean;
    showWebApps: boolean;
    snackbarText: string | undefined;
    sortOptions: [string, string][];
    sortOption: string | undefined;
    storageInfos: Record<string, StorageInfo>;
  } {
    return {
      appstoreAppInfo: undefined,
      appstoreHolochainAppInfo: undefined,
      appWebsocket: undefined,
      devHubAppInfo: null,
      devModeEnabled: false,
      howToPublishUrl:
        "https://github.com/holochain/launcher#publishing-and-updating-an-app-in-the-app-store",
      snackbarText: undefined,
      reportIssueUrl: "https://github.com/holochain/launcher/issues/new",
      showDevModeDevsOnlyWarning: false,
      ignoreDevModeWarning: false,
      sortOptions: [
        [i18n.global.t('main.name'), "name"],
        [i18n.global.t('main.nameDescending'), "name descending"],
        // ["Holochain Version", "Holochain Version"],
      ],
      sortOption: undefined,
      selectedHolochainVersion: "All Versions",
      showAdvancedSettings: false,
      showHeadlessApps: true,
      showWebApps: true,
      storageInfos: {},
      refreshing: false,
      refreshTimeout: null,
      extendedAppInfos: undefined,
      selectedApp: undefined,
      selectedGuiUpdate: undefined,
      selectedGuiUpdateLocator: undefined,
      loadingText: "",
      errorText: "Unknown error occured",
    };
  },
  emits: ["show-message", "updated-ui"],
  async mounted() {
    await this.refreshAppStates();
  },
  computed: {
    devModeOn() {
      return !!this.devHubAppInfo && (isAppRunning(this.devHubAppInfo.webAppInfo.installed_app_info) || isAppPaused(this.devHubAppInfo.webAppInfo.installed_app_info))
    },
    sortedApps() {
      // if extended happ releases are not yet fetched from the DevHub to include potential
      // GUI updates, just return installedApps with guiUpdateAvailable undefined
      let sortedAppList: Array<HolochainAppInfoExtended> = this.extendedAppInfos
          ? Object.values(this.extendedAppInfos)
          : this.installedApps.map((app) => {
        return {
          webAppInfo: app.webAppInfo,
          holochainId: app.holochainId,
          holochainVersion: app.holochainVersion,
          guiUpdateAvailable: undefined,
        }
      });

      if (this.selectedHolochainVersion !== "All Versions") {
        sortedAppList = sortedAppList.filter(
          (app) => app.holochainVersion === this.selectedHolochainVersion
        );
      }

      if (this.sortOption === "name") {
        sortedAppList = sortedAppList.sort((appA, appB) =>
          appA.webAppInfo.installed_app_info.installed_app_id.localeCompare(
            appB.webAppInfo.installed_app_info.installed_app_id
          )
        );
      } else if (this.sortOption === "name descending") {
        sortedAppList = sortedAppList.sort((appA, appB) =>
          appB.webAppInfo.installed_app_info.installed_app_id.localeCompare(
            appA.webAppInfo.installed_app_info.installed_app_id
          )
        );
      } else {
        // default is alphabetical by app id
        sortedAppList = sortedAppList.sort((appA, appB) =>
          appA.webAppInfo.installed_app_info.installed_app_id.localeCompare(
            appB.webAppInfo.installed_app_info.installed_app_id
          )
        );
      }

      return sortedAppList;
    },
    noHeadlessApps(): boolean {
      return !this.sortedApps.some(
        (app) => app.webAppInfo.web_uis.default.type === "Headless"
      );
    },
    noWebApps(): boolean {
      return this.sortedApps.every(
        (app) => app.webAppInfo.web_uis.default.type === "Headless"
      );
    },
    noHolochainVersions(): boolean {
      return this.noWebApps && this.noHeadlessApps;
    },
    holochainVersions(): string[] {
      const allApps = this.installedApps;
      return uniq(allApps.map((app) => app.holochainVersion));
    },
    holochainVersionOptions(): [string, string][] {
      let allApps = this.installedApps;
      let hcVersions: [string, string][] = [[i18n.global.t('main.allVersions'), "All Versions"]];
      uniq(allApps.map((app) => app.holochainVersion)).forEach((hcVer) => {
        hcVersions.push([hcVer, hcVer]);
      });
      return hcVersions;
    },
  },
  methods: {
    isAppDisabled,
    isAppPaused,
    isAppRunning,
    prettyBytes,
    isLoading() {
      return this.$store.state.launcherStateInfo === "loading";
    },
    openConfig() {
      (this.$refs.configDialog as typeof Config).open()
    },
    openPublishAppDialog() {
      (this.$refs["publishAppDialog"] as typeof HCDialog).open();
    },
    closePublishAppDialog() {
      (this.$refs["publishAppDialog"] as typeof HCDialog).close();
    },
    closeDevModeWarning() {
      if (this.ignoreDevModeWarning) {
        window.localStorage.setItem("ignoreDevModeDevsOnlyWarning", "true");
      }
      (this.$refs["devModeDevsOnlyWarning"] as typeof HCDialog).close();
    },
    async refreshAppStates() {

      await this.$store.dispatch(ActionTypes.fetchStateInfo);

      this.devHubAppInfo = null

      await Promise.all(
        this.installedApps.map(async (app) => {
          // Check if DevHub is installed and if so store info about it locally
          if (app.webAppInfo.installed_app_info.installed_app_id === DEVHUB_APP_ID) {
            this.devHubAppInfo = app
          }

          // Store app store for later use
          if (app.webAppInfo.installed_app_info.installed_app_id === APPSTORE_APP_ID) {
            this.appstoreHolochainAppInfo = app
          }

          return this.storageInfos[app.holochainVersion] = await invoke(
            "get_storage_info",
            { holochainId: app.holochainId }
          );
        })
      );

      const holochainId = this.$store.getters["holochainIdForDevhub"];
      // connect to AppWebsocket
      const port = this.$store.getters["appInterfacePort"](holochainId);
      const appWebsocket = await AppWebsocket.connect(`ws://localhost:${port}`, 40000);
      this.appWebsocket = appWebsocket;
      const appstoreAppInfo = await appWebsocket.appInfo({
        installed_app_id: APPSTORE_APP_ID,
      });
      this.appstoreAppInfo = appstoreAppInfo;

      await this.refreshExtendedAppInfos();
    },
    async refreshExtendedAppInfos() {
      const extendedAppInfos: Record<InstalledAppId, HolochainAppInfoExtended> = {};

      // TODO: do i need this here?
      this.installedApps.forEach((app) => {
        extendedAppInfos[app.webAppInfo.installed_app_info.installed_app_id] = {
          webAppInfo: app.webAppInfo,
          holochainId: app.holochainId,
          holochainVersion: app.holochainVersion,
          guiUpdateAvailable: undefined,
        }
      });

      this.extendedAppInfos = extendedAppInfos;

      await this.checkForUiUpdates();
    },
    async handleInstallDevHub() {
      if (this.ignoreDevModeWarning) {
        window.localStorage.setItem("ignoreDevModeDevsOnlyWarning", "true");
      }
      (this.$refs["devModeDevsOnlyWarning"] as typeof HCDialog).close();
      this.loadingText = "Installing DevHub...";
      (this.$refs.downloading as typeof HCLoading).open();

      try {
        await invoke("install_devhub", {});
        (this.$refs.downloading as typeof HCLoading).close();
        this.loadingText = "";
        await this.refreshAppStates();
      } catch (e) {
        alert(`Failed to install DevHub: ${JSON.stringify(e)}`);
        console.error(`Failed to install DevHub: ${JSON.stringify(e)}`);
        (this.$refs.downloading as typeof HCLoading).close();
        this.loadingText = "";
      }
    },
    async openApp(app: HolochainAppInfo) {
      const appId = app.webAppInfo.installed_app_info.installed_app_id;
      try {
        await invoke("open_app_ui", { appId, holochainId: app.holochainId });
        this.showMessage(`App ${appId} opened`);
      } catch (e) {
        const error = `Error opening app ${appId}: ${JSON.stringify(e)}`;
        this.showMessage(error);
        await invoke("log", {
          log: error,
        });
      }
    },
    async disableApp(app: HolochainAppInfo) {
      const appId = app.webAppInfo.installed_app_info.installed_app_id;
      try {
        await invoke("disable_app", { appId, holochainId: app.holochainId });
        await this.refreshAppStates();
        this.showMessage(`Disabled ${appId}`);

      } catch (e) {
        const error = `Disable app ${appId} failed: ${JSON.stringify(e)}`;

        // if disabling "purportedly" fails due to being offline, ignore the error.
        if (error.includes("failed to lookup address information: Temporary failure in name resolution")) {
          this.showMessage(`Disabled ${appId}`);
        } else {
          this.showMessage(error);
        }
        await invoke("log", {
          log: error,
        });

        await this.$store.dispatch(ActionTypes.fetchStateInfo);
      }
    },
    async enableApp(app: HolochainAppInfo) {
      const appId = app.webAppInfo.installed_app_info.installed_app_id;

      try {
        await invoke("enable_app", { appId, holochainId: app.holochainId });
        await this.refreshAppStates();
        this.showMessage(`Enabled ${appId}`);
      } catch (e) {
        const error = `Enable app ${appId} failed: ${JSON.stringify(e)}`;
        this.showMessage(error);
        await invoke("log", {
          log: error,
        });
      }
    },
    async startApp(app: HolochainAppInfo) {
      // console.log("@InstalledApps: RECEIVED REQUEST TO START APP.");
      const appId = app.webAppInfo.installed_app_info.installed_app_id;
      // console.log("@InstalledApps: @startApp: appId: ", appId);

      // StartApp is not available anymore in conductor API since 0.1.0-beta-rc.4: https://github.com/holochain/holochain/blob/develop/crates/holochain_conductor_api/CHANGELOG.md#010-beta-rc4
      // instead disable app followed by enable app:
      try {
        // console.log("@InstalledApps: @startApp: disabling app.");

        await invoke("disable_app", { appId, holochainId: app.holochainId });
        // console.log("@InstalledApps: @startApp: app disabled, enabling app.");

        await invoke("enable_app", { appId, holochainId: app.holochainId });
        // console.log("@InstalledApps: @startApp: app enabled.");

        await this.refreshAppStates();

        this.showMessage(`Started ${appId}`);
      } catch (e) {
        const error = `Start app ${appId} failed: ${JSON.stringify(e)}`;
        console.error(error);
        this.showMessage(error);
        await invoke("log", {
          log: error,
        });

        await this.$store.dispatch(ActionTypes.fetchStateInfo);
      }
    },
    async uninstallApp(app: HolochainAppInfo) {

      const appId = app.webAppInfo.installed_app_info.installed_app_id;

      try {
        await invoke("uninstall_app", { appId, holochainId: app.holochainId });
        await this.refreshAppStates();
        this.showMessage(`Uninstalled ${appId}`);
      } catch (e) {
        const error = `Uninstall app ${appId} failed: ${JSON.stringify(e)}`;
        this.showMessage(error);
        await invoke("log", {
          log: error,
        });
      }
    },
    async toggleDevMode() {
      // TODO: track devModeEnabled in Tauri so it can be used all over the app?
      if (this.devModeOn) {
        await this.disableApp(this.devHubAppInfo as HolochainAppInfo);
      } else {
        if (!this.devHubAppInfo) {
          // if the DevMode is requested to be turned on for the first time,
          // show a warning dialog that this is intended for developers

          if (!window.localStorage.ignoreDevModeDevsOnlyWarning) {
            (this.$refs["devModeDevsOnlyWarning"] as typeof HCDialog).open();
            return false;
          }
        }
        this.enableApp(this.devHubAppInfo as HolochainAppInfo);
      }
    },
    async reportIssue() {
      await invoke("open_url_cmd", {
        url: this.reportIssueUrl,
      });
    },
    isAppHeadless(app: HolochainAppInfo) {
      return app.webAppInfo.web_uis.default.type === "Headless";
    },
    async checkForUiUpdates() {
      console.log("Checking for UI updates...");
      // check for GUI updates
      const allApps: Array<HolochainAppInfo> = this.$store.getters["allApps"];

      const updatableApps = allApps.filter((app) => app.webAppInfo.happ_release_info?.resource_locator);

      // sort all happ release ResourceLocators by DnaHash of the DevHub they originate from
      const updatableAppsByLocatorDna: Record<DnaHashB64, HolochainAppInfo[]> = {};

      updatableApps.forEach((app) => {
        const dnaHash = app.webAppInfo.happ_release_info!.resource_locator!.dna_hash;
        const apps = updatableAppsByLocatorDna[dnaHash];

        if (apps) {
          updatableAppsByLocatorDna[dnaHash] = [...apps, app]
        } else {
          updatableAppsByLocatorDna[dnaHash] = [app!]
        }
      });

      await Promise.allSettled(Object.values(updatableAppsByLocatorDna).map(async (apps) => {
        const entryHashes = apps.map((app) => decodeHashFromBase64(app.webAppInfo.happ_release_info!.resource_locator!.resource_hash));
        const devHubDnaHash = decodeHashFromBase64(apps[0].webAppInfo.happ_release_info!.resource_locator!.dna_hash);

        try {
          console.log("@checkForUiPudates: entryHashes: ", entryHashes.map((eh) => encodeHashToBase64(eh)));
          const happReleases: Array<HappReleaseEntry | undefined> = await getHappReleasesByEntryHashes((this.appWebsocket! as AppWebsocket), this.appstoreAppInfo!, devHubDnaHash, entryHashes);

          apps.forEach((app, idx) => {
            if (happReleases[idx]) {
              console.log("official_gui: ", happReleases[idx]!.official_gui ? encodeHashToBase64(happReleases[idx]!.official_gui!) : undefined)
            }

            // if it's installed as a webapp and the happ release has an official GUI, check whether it's a new GUI
            if (app.webAppInfo.web_uis.default.type === "WebApp" && happReleases[idx]?.official_gui) {
              const guiReleaseInfo = app.webAppInfo.web_uis.default.gui_release_info;
              const guiReleaseHash = app.webAppInfo.web_uis.default.gui_release_info?.resource_locator!.resource_hash;
              console.log("guiReleaseHash: ", guiReleaseHash);
              if (guiReleaseInfo && guiReleaseHash) {
                if(guiReleaseHash != encodeHashToBase64(happReleases[idx]!.official_gui!)) {
                  this.extendedAppInfos![app.webAppInfo.installed_app_info.installed_app_id].guiUpdateAvailable = {
                    dna_hash: devHubDnaHash,
                    resource_hash: happReleases[idx]!.official_gui!,
                  }
                }
              }
            }
          })

        } catch (e) {
          console.error(`Failed to get happ releases from DevHub host of network with DNA hash ${encodeHashToBase64(devHubDnaHash)}: ${JSON.stringify(e)}`);
        }

      }))

    },
    async openUpdateGuiDialog(app: HolochainAppInfoExtended) {
      this.selectedApp = app;

      // console.log("Gui release hash @openUpdateGuiDialog: ", app.guiUpdateAvailable);
      (this.$refs.updateGuiDialog as typeof HCGenericDialog).open();

      if (this.appWebsocket && this.appstoreAppInfo) {
          const cells = appstoreCells(this.appstoreAppInfo);
        //   const guiReleaseResponse = await this.appWebsocket?.callZome({
        //   cap_secret: null,
        //   cell_id: getCellId(cells.happs.find((c) => "provisioned" in c )!)!,
        //   fn_name: "get_gui_release",
        //   zome_name: "happ_library",
        //   payload: {
        //     id: app.guiUpdateAvailable,
        //   },
        //   provenance: getCellId(cells.happs.find((c) => "provisioned" in c )!)![1],
        // });

        const guiReleaseResponse = await fetchGuiReleaseEntry(this.appWebsocket as AppWebsocket, this.appstoreAppInfo, app.guiUpdateAvailable!);

        this.selectedGuiUpdate = guiReleaseResponse.content;
        this.selectedGuiUpdateLocator = app.guiUpdateAvailable;
        console.log("Got GUI Release: ", guiReleaseResponse.content);
      } else {
        alert!("Error: AppWebsocket or Appstore AppInfo undefined.")
        this.selectedGuiUpdate = undefined;
        this.selectedGuiUpdateLocator = undefined;
      }
    },
    storageFractions(holochainVersion: string) {
      const storageInfo: StorageInfo = this.storageInfos[holochainVersion];
      if (storageInfo) {
        const totalStorage = this.totalStorage(holochainVersion);
        const fractions = Object.values(storageInfo).map(
          (value: number) => (value / totalStorage!) * 100
        );
        return fractions;
      } else {
        return undefined;
      }
    },
    totalStorage(holochainVersion: string): number | undefined {
      const storageInfo = this.storageInfos[holochainVersion];
      if (storageInfo) {
        return Object.values(storageInfo).reduce(
          (acc, currValue) => acc + currValue
        );
      } else {
        return undefined;
      }
    },
    storageLabels(holochainVersion: string) {
      const storageInfo = this.storageInfos[holochainVersion];
      if (storageInfo) {
        return Object.entries(storageInfo).map(
          ([key, value]) => `${key} (${prettyBytes(value)})`
        );
      } else {
        return undefined;
      }
    },
    async refreshStorageInfo() {
      this.refreshing = true;
      this.refreshTimeout = window.setTimeout(
        () => (this.refreshing = false),
        200
      );
      await Promise.all(
        this.installedApps.map(async (app) => {
          this.storageInfos[app.holochainVersion] = await invoke(
            "get_storage_info",
            { holochainId: app.holochainId }
          );
        })
      );
    },
    totalStorageString(hcVersion: string) {
      const totalStorageBytes = this.totalStorage(hcVersion);
      if (totalStorageBytes) {
        return prettyBytes(totalStorageBytes);
      } else {
        return "?";
      }
    },
    async updateGui() {
      this.loadingText = "Connecting with DevHub";
      (this.$refs.downloading as typeof HCLoading).open();

      this.loadingText = "fetching UI from peer host...";

      try {
        const holochainId = this.$store.getters["holochainIdForDevhub"];
        const port = this.$store.getters["appInterfacePort"](holochainId);

        const appstoreAppInfo = this.appstoreAppInfo;

        if (appstoreAppInfo) {

          await tryWithHosts<void>(
            async (host) => {

              const bytes = await invoke("fetch_gui", {
                appPort: port,
                appstoreAppId: appstoreAppInfo!.installed_app_id,
                host: Array.from(host),
                devhubHappLibraryDnaHash: Array.from(this.selectedGuiUpdateLocator!.dna_hash), // DNA hash of the DevHub to which the remote call shall be made
                appstorePubKey: encodeHashToBase64(appstoreAppInfo!.agent_pub_key),
                guiReleaseHash: this.selectedGuiUpdate ? encodeHashToBase64(this.selectedGuiUpdate.web_asset_id) : undefined,
              });

              this.loadingText = "Installing...";

              await invoke("update_default_ui", {
                holochainId: this.selectedApp!.holochainId,
                appId: this.selectedApp!.webAppInfo.installed_app_info.installed_app_id,
                uiZipBytes: bytes,
                guiReleaseInfo: {
                  resource_locator: locatorToLocatorB64(this.selectedApp!.guiUpdateAvailable!),
                  version: this.selectedGuiUpdate?.version,
                },
              });

              this.loadingText = "";
              (this.$refs.downloading as typeof HCLoading).close();
              (this.$refs.updateGuiDialog as typeof HCGenericDialog).close();
              this.selectedGuiUpdate = undefined;
              this.selectedGuiUpdateLocator = undefined;

              // to remove the update button:
              await this.refreshAppStates();
              this.checkForUiUpdates();

            },
            this.appWebsocket as AppWebsocket,
            appstoreAppInfo!,
            this.selectedGuiUpdateLocator!.dna_hash,
            "happ_library",
            "get_webasset",
          );

          this.$emit('updated-ui');

        } else {
          console.error("Error updating the UI: Undefined appstoreAppInfo");
          this.showMessage(`Error updating the UI: Undefined appstoreAppInfo`);
          (this.$refs.downloading as typeof HCLoading).close();
          this.loadingText = "";
        }
      } catch (e) {
        console.error("Error updating the UI: ", e);
        this.showMessage(`Error fetching the UI: ${e}`);
        (this.$refs.downloading as typeof HCLoading).close();
        this.loadingText = "";
      }
    },
    showMessage(message: string) {
      this.$emit("show-message", message);
    },
  },
});
</script>
<!-- We don't have scoped styles with classes because it becomes harder to export a reusable library -->

<style scoped>
h2 {
  font-weight: 600;
  font-size: 1.2em;
  margin: 0;
}

.section {
  align-items: center;
}

.section-title {
  margin: 10px 0 10px 10px;
  padding-bottom: 3px;
  align-items: center;
  font-size: 23px;
  color: rgba(0, 0, 0, 0.6);
}

.section-container {
  border-radius: 15px;
  background-color: white;
  padding: 15px;
  box-shadow: 0 0px 5px #9b9b9b;
  margin-bottom: 20px;
}

.hc-version {
  align-items: center;
  flex: 1;
  margin-top: 8px;
  padding: 8px 0;
}


</style>
