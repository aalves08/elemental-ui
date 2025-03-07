<script>
import Loading from '@shell/components/Loading';
import LabeledSelect from '@shell/components/form/LabeledSelect';
import { Banner } from '@components/Banner';
import AsyncButton from '@shell/components/AsyncButton';
import { randomStr, CHARSET } from '@shell/utils/string';
import { ELEMENTAL_SCHEMA_IDS } from '../config/elemental-types';
import {
  getOperatorVersion,
  checkGatedFeatureCompatibility,
  BUILD_MEDIA_RAW_SUPPORT,
  BUILD_MEDIA_ARCH_SUPPORT,
  CHANNEL_NO_LONGER_IN_SYNC,
  ALL_AREAS,
  ALL_MODES,
} from '../utils/feature-versioning';

export const MEDIA_TYPES = {
  RAW: {
    filterType: 'container',
    type:       'raw',
    label:      'Raw',
    extension:  'img'
  },
  ISO: {
    type:      'iso',
    label:     'Iso',
    extension: 'iso'
  }
};

const SPECIAL_DELIMITATOR = ':::::';

export default {
  name:       'BuildMedia',
  components: {
    Loading,
    LabeledSelect,
    Banner,
    AsyncButton
  },
  props: {
    displayRegEndpoints: {
      type:    Boolean,
      default: true
    },
    registrationEndpointList: {
      type:    Array,
      default: () => []
    },
    registrationEndpoint: {
      type:    String,
      default: ''
    },
    resource: {
      type:    String,
      default: ''
    },
    mode: {
      type:    String,
      default: ''
    }
  },
  async fetch() {
    this.seedImagesList = await this.$store.dispatch('management/findAll', { type: ELEMENTAL_SCHEMA_IDS.SEED_IMAGE });
    this.managedOsVersions = await this.$store.dispatch('management/findAll', { type: ELEMENTAL_SCHEMA_IDS.MANAGED_OS_VERSIONS });

    this.operatorVersion = await getOperatorVersion(this.$store);

    // default to ISO, which is always present, which will trigger the watcher
    this.buildMediaTypeSelected = MEDIA_TYPES.ISO.type;
  },
  data() {
    return {
      seedImagesList:               [],
      managedOsVersions:            [],
      filteredManagedOsVersions:    [],
      operatorVersion:              '',
      buildMediaOsVersions:         [],
      buildMediaTypes:              [
        { label: MEDIA_TYPES.ISO.label, value: MEDIA_TYPES.ISO.type },
        { label: MEDIA_TYPES.RAW.label, value: MEDIA_TYPES.RAW.type },
      ],
      buildMediaTypeSelected:       '',
      buildMediaOsVersionSelected:  '',
      registrationEndpointSelected: '',
      mediaBuildTriggerError:       '',
      seedImage:                    undefined,
      buildBtnCallback:             undefined
    };
  },
  watch: {
    buildMediaTypeSelected(neu) {
      if (neu) {
        this.buildMediaOsVersionSelected = '';
        const selectedFilterType = neu === MEDIA_TYPES.ISO.type ? MEDIA_TYPES.ISO.type : MEDIA_TYPES.RAW.filterType;

        this.filteredManagedOsVersions = this.managedOsVersions.filter(v => v.spec?.type === selectedFilterType) || [];
        const buildMediaOsVersions = [];

        this.filteredManagedOsVersions.forEach((osVersion) => {
          if (this.supportBuildMediaArchitecture && osVersion.spec?.metadata?.platforms?.length) {
            osVersion.spec?.metadata?.platforms.forEach((arch) => {
              buildMediaOsVersions.push({
                label: `${ osVersion.spec?.metadata?.displayName } ${ osVersion.spec?.version } - ${arch}${ this.supportChannelNoLongerInSync && typeof osVersion.inSync === 'boolean' && !osVersion.inSync ? ' - (deprecated)' : '' }`,
                value: `${neu === MEDIA_TYPES.ISO.type ? osVersion.spec?.metadata?.uri : osVersion.spec?.metadata?.upgradeImage}${SPECIAL_DELIMITATOR}${arch}`,
              });
            });
          } else {
            buildMediaOsVersions.push({
              label: `${ osVersion.spec?.metadata?.displayName } ${ osVersion.spec?.version } ${ this.supportChannelNoLongerInSync && typeof osVersion.inSync === 'boolean' && !osVersion.inSync ? '(deprecated)' : '' }`,
              value: neu === MEDIA_TYPES.ISO.type ? osVersion.spec?.metadata?.uri : osVersion.spec?.metadata?.upgradeImage,
            });
          }
        });

        this.buildMediaOsVersions = buildMediaOsVersions;
      }
    }
  },
  computed: {
    supportChannelNoLongerInSync() {
      return checkGatedFeatureCompatibility(ALL_AREAS, ALL_MODES, CHANNEL_NO_LONGER_IN_SYNC, this.operatorVersion);
    },
    supportBuildMediaArchitecture() {
      return checkGatedFeatureCompatibility(this.resource, this.mode, BUILD_MEDIA_ARCH_SUPPORT, this.operatorVersion);
    },
    isRawDiskImageBuildSupported() {
      const check = checkGatedFeatureCompatibility(this.resource, this.mode, BUILD_MEDIA_RAW_SUPPORT, this.operatorVersion);

      if (!check) {
        this.buildMediaTypeSelected = MEDIA_TYPES.ISO.type; // eslint-disable-line vue/no-side-effects-in-computed-properties
      }

      return check;
    },
    registrationEndpointsOptions() {
      const activeRegEndpoints = this.registrationEndpointList.filter(item => item.state === 'active');

      return activeRegEndpoints.map((machReg) => {
        return {
          label: machReg.name,
          value: `${ machReg.namespace }/${ machReg.name }`
        };
      });
    },
    isBuildMediaBtnEnabled() {
      if (this.displayRegEndpoints) {
        return this.isRawDiskImageBuildSupported ? this.registrationEndpointSelected && this.buildMediaOsVersionSelected && this.buildMediaTypeSelected : this.registrationEndpointSelected && this.buildMediaOsVersionSelected;
      }

      return this.isRawDiskImageBuildSupported ? this.buildMediaOsVersionSelected && this.buildMediaTypeSelected : this.buildMediaOsVersionSelected;
    },
    seedImageFound() {
      if (this.seedImage) {
        const imgFound = this.seedImagesList.find(img => img.metadata?.uid === this.seedImage.metadata?.uid);

        if (imgFound) {
          return imgFound;
        }
      }

      return {};
    },
    downloadUrl() {
      return this.seedImageFound?.status?.downloadURL;
    },
    checksumUrl() {
      return this.seedImageFound?.status?.checksumURL;
    },
    isMediaBuilt() {
      if (this.seedImageFound && this.seedImageFound.status?.downloadURL) {
        this.buildBtnCallback(true);

        return true;
      }

      return false;
    },
    mediaBuildProcessError() {
      if (this.seedImageFound && this.seedImageFound.status?.conditions) {
        let errorFound = '';
        const excludedReasons = ['SeedImageBuildSuccess', 'ResourcesSuccessfullyCreated', 'SeedImageBuildOngoing'];

        for (let i = 0; i < this.seedImageFound.status?.conditions.length; i++) {
          const condition = this.seedImageFound.status?.conditions[i];

          if (condition.status === 'False' && !excludedReasons.includes(condition.reason)) {
            this.buildBtnCallback(false);
            errorFound = condition.message;
            break;
          }
        }

        if (errorFound) {
          return errorFound;
        }
      }

      return '';
    }
  },
  methods: {
    async buildMedia(btnCb) {
      this.mediaBuildTriggerError = '';
      this.seedImage = undefined;
      const machineRegName = this.displayRegEndpoints ? this.registrationEndpointSelected.split('/')[1] : this.registrationEndpoint.split('/')[1];
      const machineRegNamespace = this.displayRegEndpoints ? this.registrationEndpointSelected.split('/')[0] : this.registrationEndpoint.split('/')[0];

      let targetPlatform = '';
      let baseImage = this.buildMediaOsVersionSelected;

      if (this.supportBuildMediaArchitecture && this.buildMediaOsVersionSelected.includes(SPECIAL_DELIMITATOR)) {
        targetPlatform = this.buildMediaOsVersionSelected.split(SPECIAL_DELIMITATOR)[1];
        baseImage = this.buildMediaOsVersionSelected.split(SPECIAL_DELIMITATOR)[0];
      }

      const seedImageObject = {
        metadata: {
          name:      `media-image-reg-${ machineRegName }-${ randomStr(8, CHARSET.ALPHA_LOWER ) }`,
          namespace: 'fleet-default'
        },
        spec: {
          baseImage,
          registrationRef: {
            name:      machineRegName,
            namespace: machineRegNamespace
          }
        },
        type: ELEMENTAL_SCHEMA_IDS.SEED_IMAGE,
      };

      if (this.isRawDiskImageBuildSupported) {
        seedImageObject.spec.type = this.buildMediaTypeSelected;
      }

      if (targetPlatform) {
        seedImageObject.spec.targetPlatform = targetPlatform;
      }

      const seedImageModel = await this.$store.dispatch('management/create', seedImageObject);

      try {
        this.seedImage = await seedImageModel.save({ url: `/v1/${ ELEMENTAL_SCHEMA_IDS.SEED_IMAGE }`, method: 'POST' });
        this.buildBtnCallback = btnCb;
      } catch (e) {
        this.mediaBuildTriggerError = e;
        btnCb(false);
      }
    }
  },
};
</script>

<template>
  <Loading v-if="$fetchState.pending" />
  <div v-else>
    <h3 class="build-iso-title">
      {{ t('elemental.machineRegistration.edit.buildMediaTitle') }}
    </h3>
    <div class="row mb-15">
      <div
        v-if="displayRegEndpoints"
        class="col span-2"
      >
        <LabeledSelect
          v-model:value="registrationEndpointSelected"
          class="mr-20"
          data-testid="select-registration-endpoint-build-media"
          :label="t('elemental.machineRegistration.create.machineReg')"
          :placeholder="t('elemental.dashboard.regEndpointPlaceholder')"
          :options="registrationEndpointsOptions"
        />
      </div>
      <div
        v-if="isRawDiskImageBuildSupported"
        class="col span-2"
      >
        <LabeledSelect
          v-model:value="buildMediaTypeSelected"
          class="mr-20"
          data-testid="select-media-type-build-media"
          :label="t('elemental.machineRegistration.edit.mediaType')"
          :placeholder="t('elemental.machineRegistration.edit.mediaTypePlaceholder')"
          :options="buildMediaTypes"
          option-key="value"
        />
      </div>
      <div class="col span-3">
        <LabeledSelect
          v-model:value="buildMediaOsVersionSelected"
          class="mr-20"
          data-testid="select-os-version-build-media"
          :label="t('elemental.machineRegistration.edit.osVersion')"
          :disabled="!buildMediaTypeSelected"
          :placeholder="t('elemental.machineRegistration.edit.osVersionPlaceholder')"
          :options="buildMediaOsVersions"
          option-key="value"
        />
      </div>
      <div class="col mt-10 span-5 flex">
        <AsyncButton
          mode="buildMedia"
          class="mr-20"
          data-testid="build-media-btn"
          :disabled="(!isBuildMediaBtnEnabled || isMediaBuilt) ? 'disabled' : null"
          @click="buildMedia"
        />
        
        <div class="download-group">
          <a
            :disabled="!isMediaBuilt ? 'disabled' : null"
            class="btn role-primary"
            data-testid="download-media-btn"
            target="_blank"
            rel="noopener nofollow"
            :href="downloadUrl"
          >
            {{ t('elemental.machineRegistration.edit.downloadMedia') }}
          </a>  
          <a 
            v-if="checksumUrl"
            data-testid="download-checksum-btn" 
            class="checksum-btn"
            target="_blank"
            rel="noopener nofollow"
            download="checksum.sh256"
            :href="checksumUrl">{{ t('elemental.machineRegistration.edit.downloadChecksum') }}</a>
        </div>
      </div>
    </div>
    <div class="row mb-10">
      <div class="col span-9">
        <p v-clean-html="t('elemental.machineRegistration.edit.userWarning',{}, true)" class="user-warn">
        </p>
      </div>
    </div>
    <Banner
      v-if="mediaBuildTriggerError || mediaBuildProcessError"
      color="error"
      data-testid="build-media-banner"
    >
      <div v-clean-html="mediaBuildTriggerError || mediaBuildProcessError" />
    </Banner>
  </div>
</template>

<style lang="scss" scoped>
.user-warn {
  font-size: 13px;
  color: var(--darker);
}

.download-group {
  max-width: fit-content;
  display: inline-flex;
  flex-direction: column;
  vertical-align: top;

  .checksum-btn {
    cursor: pointer;
    margin-top: 4px;
    font-size: 12px;
    text-align: center;
    text-decoration: underline;
  }
}
</style>
