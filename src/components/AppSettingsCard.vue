<template>
  <HCGenericDialog
    @confirm="uninstallApp(app)"
    closeOnSideClick
    ref="uninstall-app-dialog"
    :primaryButtonLabel="uninstalling ? $t('buttons.uninstalling') : $t('buttons.uninstall')"
    :primaryButtonDisabled="uninstalling"
    ><div style="text-align: center">
      {{ $t('dialogs.confirmUninstallApp') }}
    </div>
  </HCGenericDialog>


  <HCGenericDialog
    @confirm="uninstallApp(app);"
    @closing="confirmUninstallDevHub = false"
    closeOnSideClick
    ref="uninstall-devhub-dialog"
    :primaryButtonLabel="uninstalling ? $t('buttons.uninstalling') : $t('buttons.uninstall')"
    :primaryButtonDisabled="!confirmUninstallDevHub || uninstalling"
  >
    <div style="text-align: center; padding: 0 20px;">
      <h1 style="margin-top: -10px;">{{ $t('dialogs.warning') }}</h1>
      <div style="text-align: left; margin-top: 40px;">{{ $t('dialogs.confirmUninstallDevHub.text') }}</div>
      <div class="row" style="margin-top: 40px; margin-left: 10px;">
        <ToggleSwitch
          :sliderOn="confirmUninstallDevHub"
          @click="confirmUninstallDevHub = !confirmUninstallDevHub"
          @keydown.enter="confirmUninstallDevHub = !confirmUninstallDevHub"
        ></ToggleSwitch>
        <div style="text-align: left; margin-left: 20px;">{{ $t('dialogs.confirmUninstallDevHub.confirmation') }}</div>
      </div>
    </div>
  </HCGenericDialog>

  <div class="container">
    <div
      style="
        position: relative;
        display: flex;
        flex-direction: row;
        align-items: center;
        width: 100%;
        height: 110px;
      "
    >
    <!-- App Logo with Holo Identicon -->
      <div style="position: relative">
        <img
          v-if="app.webAppInfo.icon_src"
          class="appIcon"
          :src="`${app.webAppInfo.icon_src}`"
        />
        <div
          v-else
          class="appIcon column center-content"
          style="background-color: #372ba5"
        >
          <div style="color: white; font-size: 45px; font-weight: 600">
            {{ app.webAppInfo.installed_app_info.installed_app_id.slice(0, 2) }}
          </div>
        </div>
      </div>
      <!-- ------------- -->

      <!-- Installed App Id -->
      <div
        style="
          display: flex;
          font-size: 23px;
          font-weight: 700;
          margin-left: 40px;
          margin-right: 30px;
          word-break: break-all;
        "
      >
        {{ app.webAppInfo.installed_app_info.installed_app_id }}
      </div>
      <!-- ---------------- -->

      <!-- App status indicator -->
      <sl-tooltip
        style="--show-delay: 500"
        hoist
        placement="top"
        :content="getAppStatus(app)"
      >
        <div
          :class="{
            running: isAppRunning(app.webAppInfo.installed_app_info) || isAppPaused(app.webAppInfo.installed_app_info),
            stopped: isAppDisabled(app.webAppInfo.installed_app_info),
            paused: false,
          }"
          class="app-status"
          style="margin-right: 29px"
          tabindex="0"
        ></div>
      </sl-tooltip>
      <!-- ----------------- -->

      <!-- spacer -->
      <div style="flex: 1;"></div>

      <!-- GUI update available Icon -->
      <div
        v-if="
          app.guiUpdateAvailable
        "
        style="display: flex; position: relative;"
      >
        <sl-tooltip class="tooltip" hoist placement="top" content="New UI available">
          <!-- <img
            tabindex="0"
            style="margin-right: 29px; width: 24px; cursor: pointer"
            src="/img/Open_App.svg"
            @click="$emit('openApp', app)"
            v-on:keyup.enter="$emit('openApp', app)"
          /> -->
          <div
            @click="$emit('updateGui', app)"
            @keypress.enter="$emit('updateGui', app)"
            tabindex="0"
            class="update-button"
          >
            Update
          </div>
          <div style="background: rgb(255, 217, 0); border-radius: 50%; height: 15px; width: 15px; position: absolute; bottom: 20px; right: 22px;"></div>
        </sl-tooltip>
      </div>
      <!-- -------------------- -->

      <!-- Open App Icon Button -->
      <div
        v-if="
          (isAppRunning(app.webAppInfo.installed_app_info) || isAppPaused(app.webAppInfo.installed_app_info)) && !isAppHeadless(app)
        "
        style="display: flex"
      >
        <sl-tooltip class="tooltip" hoist placement="top" content="Open App">
          <img
            tabindex="0"
            style="margin-right: 29px; width: 24px; cursor: pointer"
            src="/img/Open_App.svg"
            @click="$emit('openApp', app)"
            v-on:keyup.enter="$emit('openApp', app)"
          />
        </sl-tooltip>
      </div>
      <!-- ------------------- -->

      <!-- Disable/enable switch -->
      <sl-tooltip
        class="tooltip"
        hoist
        placement="top"
        :content="
          isAppRunning(app.webAppInfo.installed_app_info)
            ? 'Disable App'
            : 'Enable App'
        "
      >
        <ToggleSwitch
          v-if="
            isAppUninstallable(
              app.webAppInfo.installed_app_info.installed_app_id
            )
          "
          style="margin-right: 29px"
          :sliderOn="isSliderOn"
          @click="handleSlider(app)"
          @keydown.enter="handleSlider(app)"
        />
      </sl-tooltip>
      <!-- ------------------- -->

      <!-- Triple dot icon to show app details -->
      <sl-tooltip class="tooltip" hoist placement="top" content="App Details">
        <HCMoreToggle
          @toggle="showMore = !showMore"
          style="margin-right: 33px"
          tabindex="0"
        />
      </sl-tooltip>
      <!-- ------------------- -->
    </div>

    <!-------------- App details --------------->
    <div
      v-if="showMore"
      class="column appDetails"
      style="align-items: left; margin-bottom: 20px; padding-left: 115px;"
    >
      <div class="row">
        <span style="margin-right: 10px; font-weight: bold; font-size: 1em"
          >{{ $t('main.holochainVersion') }}:</span
        >
        <span style="opacity: 0.7; font-family: monospace: font-size: 1em;">{{
          app.holochainId.type === "CustomBinary"
            ? "Custom Binary"
            : app.holochainId.content
        }}</span>
        <!-- <span style="flex: 1;"></span>
        <img
          src="/img/refresh.png"
          title="Refresh"
          @click="refresh"
          style="width: 20px; height: 20px; margin-right: 30px; cursor: pointer;"
        > -->
      </div>

      <!-- Public Key -->
      <div class="row" style="align-items: center;">
        <!-- assumes same agent pub key for all cells (just taking the first one) -->
        <!-- <div v-show="showPubKeyTooltip" class="tooltip">Copied!</div> -->
        <span style="margin-right: 10px; font-weight: bold; font-size: 1em; vertical-align: middle;"
          >{{ $t('settings.publicKey') }}:</span
        >
        <sl-tooltip class="tooltip" hoist placement="top" :content="showPubKeyTooltip ? $t('main.copied') : $t('main.yourPublicKey')">
          <HoloIdenticon
            class="holoIdenticon"
            :hash="getPubKey()"
            :size="42"
            tabindex="0"
            @click="copyPubKey()"
            @keypress.enter="copyPubKey()"
          ></HoloIdenticon>
        </sl-tooltip>
      </div>

      <!-- provisioned cells -->
      <div
        class="row"
        style="margin-right: 30px"
      >
        <span style="margin-right: 10px; font-weight: bold; font-size: 1em"
          >Provisioned Cells:</span
        ><span style="display: flex; flex: 1">{{ provisionedCells.length }}</span>
        <span
          style="opacity: 0.7; cursor: pointer; font-size: 0.8em"
          @click="showProvisionedCells = !showProvisionedCells"
          >{{ showProvisionedCells ? `[${$t('main.hide')}]` : `[${$t('main.show')}]` }}
        </span>
      </div>
      <div v-if="showProvisionedCells" style="margin-right: 20px">
        <InstalledCellCard
          v-for="[roleName, cellInfo] in provisionedCells"
          :key="roleName"
          style="margin: 12px 0"
          :cellInfo="cellInfo"
          :roleName="roleName"
          :holochainId="app.holochainId"
        >
        </InstalledCellCard>
      </div>

      <!-- enabled cloned cells -->
      <div
        class="row"
        style="margin-top: 20px; margin-right: 30px"
      >
        <span style="margin-right: 10px; font-weight: bold; font-size: 1em"
          >Cloned Cells:</span
        ><span style="display: flex; flex: 1">{{ enabledClonedCells.length }}</span>
        <span
          style="opacity: 0.7; cursor: pointer; font-size: 0.8em"
          @click="showClonedCells = !showClonedCells"
          >{{ showClonedCells ? `[${$t('main.hide')}]` : `[${$t('main.show')}]` }}
        </span>
      </div>
      <div
        v-if="showClonedCells"
        style="margin-right: 20px"
      >
        <div v-if="enabledClonedCells.length > 0">
          <InstalledCellCard
            v-for="[roleName, cellInfo] in enabledClonedCells"
            :key="roleName"
            :cellInfo="cellInfo"
            :roleName="roleName"
            :holochainId="app.holochainId"
          >
          </InstalledCellCard>
        </div>

        <div v-else style="text-align: center; opacity: 0.7">
          {{ $t("main.noClonedCells") }}
        </div>
      </div>


      <!-- disabled cloned cells -->
      <div
        class="row"
        style="margin-top: 20px; margin-right: 30px"
      >
        <span style="margin-right: 10px; font-weight: bold; font-size: 1em"
          >Disabled Cloned Cells:</span
        ><span style="display: flex; flex: 1">{{ disabledClonedCells.length }}</span>
        <span
          style="opacity: 0.7; cursor: pointer; font-size: 0.8em"
          @click="showDisabledClonedCells = !showDisabledClonedCells"
          >{{ showDisabledClonedCells ? `[${$t('main.hide')}]` : `[${$t('main.show')}]` }}
        </span>
      </div>
      <div
        v-if="showDisabledClonedCells"
        style="margin-right: 20px"
      >
        <div v-if="disabledClonedCells.length > 0">
          <DisabledCloneCard
            v-for="[roleName, cellInfo] in disabledClonedCells"
            :key="roleName"
            style="margin: 12px 0;"
            :cellInfo="cellInfo"
            :roleName="roleName"
            :holochainId="app.holochainId"
            :appId="app.webAppInfo.installed_app_info.installed_app_id"
          >
          </DisabledCloneCard>
        </div>

        <div v-else style="text-align: center; opacity: 0.7">
          {{ $t("main.noDisabledClonedCells") }}
        </div>
      </div>

      <span
        v-if="getReason(app.webAppInfo.installed_app_info)"
        style="margin-top: 16px;"
      >
        {{ getReason(app.webAppInfo.installed_app_info) }}
      </span>

      <div
        style="
          display: flex;
          flex-direction: row;
          justify-content: flex-end;
          margin-top: 40px;
          margin-right: 20px;
        "
      >
        <HCButton
          class="btn"
          style="--hc-primary-color: #d80d0d"
          @click="requestUninstall"
          v-if="
            isAppUninstallable(
              app.webAppInfo.installed_app_info.installed_app_id
            )
          "
          outlined
          >{{ $t("buttons.uninstall") }}
        </HCButton>

        <HCButton
          style="--hc-primary-color: #dd821a"
          v-if="
            !isAppDisabled(app.webAppInfo.installed_app_info) &&
            !isAppPaused(app.webAppInfo.installed_app_info) &&
            isAppUninstallable(
              app.webAppInfo.installed_app_info.installed_app_id
            )
          "
          outlined
          @click="disableApp(app)"
          >{{ $t("buttons.disable") }}
        </HCButton>
        <HCButton
          style="--hc-primary-color: #008704"
          v-if="isAppDisabled(app.webAppInfo.installed_app_info)"
          @click="enableApp(app)"
          outlined
          >{{ $t("buttons.enable") }}
        </HCButton>
        <HCButton
          style="--hc-primary-color: #008704;"
          v-if="false"
          @click="startApp(app)"
          outlined
          >{{ $t("buttons.start") }}
        </HCButton>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, PropType } from "vue";
import { HolochainAppInfo, HolochainAppInfoExtended } from "../types";
import { isAppRunning, isAppDisabled, isAppPaused, getReason, flattenCells, getCellId } from "../utils";
import { writeText } from "@tauri-apps/api/clipboard";
import { CellInfo, CellType, ClonedCell, encodeHashToBase64, NetworkInfo } from "@holochain/client";

import "@shoelace-style/shoelace/dist/components/tooltip/tooltip.js";
import "@shoelace-style/shoelace/dist/themes/light.css";
// import "@holochain-open-dev/utils/dist/holo-identicon";
import HoloIdenticon from "../components/subcomponents/HoloIdenticon.vue";

import ToggleSwitch from "./subcomponents/ToggleSwitch.vue";
import HCButton from "./subcomponents/HCButton.vue";
import HCMoreToggle from "./subcomponents/HCMoreToggle.vue";
import HCGenericDialog from "./subcomponents/HCGenericDialog.vue";
import InstalledCellCard from "./subcomponents/InstalledCellCard.vue";
import DisabledCloneCard from "./subcomponents/DisabledCloneCard.vue";
import { APPSTORE_APP_ID, DEVHUB_APP_ID } from "../constants";

export default defineComponent({
  name: "AppSettingsCard",
  components: {
    ToggleSwitch,
    HCButton,
    HCMoreToggle,
    HCGenericDialog,
    HoloIdenticon,
    InstalledCellCard,
    DisabledCloneCard,
  },
  props: {
    appIcon: {
      type: String,
    },
    app: {
      type: Object as PropType<HolochainAppInfoExtended>,
      required: true,
    },
  },
  data(): {
    confirmUninstallDevHub: boolean;
    showMore: boolean;
    showPubKeyTooltip: boolean;
    gossipInfo: Record<string, NetworkInfo>;
    showProvisionedCells: boolean;
    showClonedCells: boolean;
    showDisabledClonedCells: boolean;
    uninstalling: boolean;
  } {
    return {
      confirmUninstallDevHub: false,
      showMore: false,
      showPubKeyTooltip: false,
      gossipInfo: {},
      showProvisionedCells: true,
      showClonedCells: false,
      showDisabledClonedCells: false,
      uninstalling: false,
    };
  },
  emits: ["openApp", "enableApp", "disableApp", "startApp", "uninstallApp", "updateGui"],
  computed: {
    provisionedCells(): [string, CellInfo][] {
      const provisionedCells = flattenCells(this.app.webAppInfo.installed_app_info.cell_info)
        .filter(([_roleName, cellInfo]) => "provisioned" in cellInfo)
        .sort(([roleName_a, _cellInfo_a], [roleName_b, _cellInfo_b]) => roleName_a.localeCompare(roleName_b));
      return provisionedCells
    },
    enabledClonedCells(): [string, CellInfo][] {
      return flattenCells(this.app.webAppInfo.installed_app_info.cell_info)
        .filter(([_roleName, cellInfo]) => "cloned" in cellInfo)
        .filter(([_roleName, cellInfo]) => (cellInfo as { [CellType.Cloned]: ClonedCell }).cloned.enabled)
        .sort(([roleName_a, _cellInfo_a], [roleName_b, _cellInfo_b]) => roleName_a.localeCompare(roleName_b));
    },
    disabledClonedCells(): [string, CellInfo][] {
      return flattenCells(this.app.webAppInfo.installed_app_info.cell_info)
        .filter(([_roleName, cellInfo]) => "cloned" in cellInfo)
        .filter(([_roleName, cellInfo]) => !(cellInfo as { [CellType.Cloned]: ClonedCell }).cloned.enabled)
        .sort(([roleName_a, _cellInfo_a], [roleName_b, _cellInfo_b]) => roleName_a.localeCompare(roleName_b));
    },
    isSliderOn() {
      return (isAppRunning(this.app.webAppInfo.installed_app_info) || isAppPaused(this.app.webAppInfo.installed_app_info));
    },
  },
  methods: {
    encodeHashToBase64,
    getReason,
    isAppRunning,
    isAppDisabled,
    isAppPaused,
    writeText,
    getCellId,
    isAppHeadless(app: HolochainAppInfo) {
      return app.webAppInfo.web_uis.default.type === "Headless";
    },
    requestUninstall() {
      if (this.app.webAppInfo.installed_app_info.installed_app_id === DEVHUB_APP_ID) {
        (this.$refs["uninstall-devhub-dialog"] as typeof HCGenericDialog).open();
      } else {
        (this.$refs["uninstall-app-dialog"] as typeof HCGenericDialog).open();
      }
    },
    async enableApp(app: HolochainAppInfo) {
      this.$emit("enableApp", app);
    },
    async disableApp(app: HolochainAppInfo) {
      this.$emit("disableApp", app);
    },
    async startApp(app: HolochainAppInfo) {
      this.$emit("startApp", app);
    },
    async uninstallApp(app: HolochainAppInfo) {
      this.$emit("uninstallApp", app);
      this.uninstalling = true;
    },
    getAppStatus(app: HolochainAppInfo) {
      if (isAppRunning(app.webAppInfo.installed_app_info) || isAppPaused(app.webAppInfo.installed_app_info)) {
        return "Running";
      }
      if (isAppDisabled(app.webAppInfo.installed_app_info)) {
        return "Disabled";
      }
      // Currently this won't be called as paused and running are conflated both into running
      // because app status is not getting updated: https://github.com/holochain/holochain/issues/1580#issuecomment-1377471698
      if (isAppPaused(app.webAppInfo.installed_app_info)) {
        return "Offline/Paused";
      }
      return "Unknown State";
    },
    isAppUninstallable(installedAppId: string) {
      return installedAppId !== APPSTORE_APP_ID;
    },
    async handleSlider(app: HolochainAppInfo) {
      if (isAppRunning(app.webAppInfo.installed_app_info) || isAppPaused(app.webAppInfo.installed_app_info)) {
        await this.disableApp(app);
      } else if (isAppDisabled(app.webAppInfo.installed_app_info)) {
        await this.enableApp(app);
      } else if (isAppPaused(app.webAppInfo.installed_app_info)) {
        // Currently this won't be called as paused and running are conflated both into running
        // because app status is not getting updated: https://github.com/holochain/holochain/issues/1580#issuecomment-1377471698
        await this.startApp(app);
      } else {
        throw new Error("Unknown App state.");
      }
    },
    copyPubKey() {
      const pubKey =
        this.getPubKey();
      this.writeText(encodeHashToBase64(new Uint8Array(pubKey)));
      this.showPubKeyTooltip = true;
      setTimeout(() => {
        this.showPubKeyTooltip = false;
      }, 1200);
    },
    getPubKey() {
      const cell = Object.values(this.app.webAppInfo.installed_app_info.cell_info)[0]
        .find((c) => "provisioned" in c);

      if (!cell || !("provisioned" in cell)) {
        throw new Error("no provisioned cell found");
      }

      return cell.provisioned.cell_id[1];
    },
  },
});
</script>

<style scoped>
.container {
  position: relative;
  display: flex;
  flex: 1;
  flex-direction: column;
  align-items: center;
  background: #ffffff;
  width: 100%;
  border-radius: 15px;
  background-color: white;
  padding: 0 15px;
  box-shadow: 0 0px 5px #9b9b9b;
  margin-bottom: 12px;
}

.btn {
  margin: 5px;
}

.tooltip {
  --show-delay: 1000;
}

.tooltip::part(base) {
  font-family: "Poppins";
}

.appIcon {
  display: flex;
  width: 80px;
  height: 80px;
  padding: 0;
  border-radius: 12px;
  object-fit: cover;
}

.appDetails .row {
  margin-bottom: 20px;
}

.holoIdenticon {
  border-radius: 12px;
  cursor: pointer;
}

.app-status {
  height: 10px;
  width: 10px;
  border-radius: 50%;
}

.running {
  background-color: rgb(0, 185, 0);
}

.stopped {
  background-color: rgb(220, 0, 0);
}

.paused {
  background-color: rgb(175, 175, 175);
}

.tooltip {
  position: absolute;
  /* color: #482edf; */
  color: white;
  bottom: 56px;
  left: 62px;
  background: #5537fc;
  border-radius: 5px;
  /* border: 2px solid #482edf; */
  padding: 1px 7px;
}

.update-button {
  font-weight: bold;
  color: black;
  cursor: pointer;
  border: 2px solid black;
  border-radius: 4px;
  padding: 0 5px;
  margin-right: 29px;
  opacity: 0.85;
}

.update-button:hover {
  opacity: 0.6;
}

</style>
