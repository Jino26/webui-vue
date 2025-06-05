<template>
  <div>
    <div class="form-background p-3">
      <b-form @submit.prevent="onSubmitUpload">
        <b-form-group
          v-if="isTftpUploadAvailable && tftpServer"
          :label="$t('pageFirmware.form.updateFirmware.fileSource')"
          :disabled="isPageDisabled"
        >
          <b-form-radio v-model="isWorkstationSelected" :value="true">
            {{ $t('pageFirmware.form.updateFirmware.workstation') }}
          </b-form-radio>
          <b-form-radio v-model="isWorkstationSelected" :value="false">
            {{ $t('pageFirmware.form.updateFirmware.tftpServer') }}
            <span
              ><info-tooltip
                class="info-icon"
                :title="$t('pageFirmware.form.updateFirmware.tftpServerInfo')"
            /></span>
          </b-form-radio>
        </b-form-group>

        <!-- Workstation Upload -->
        <template v-if="isWorkstationSelected">
          <b-form-group
            :label="$t('pageFirmware.form.updateFirmware.imageFile')"
            label-for="image-file"
          >
            <form-file
              id="image-file"
              :disabled="isPageDisabled"
              accept=".tar"
              :state="getValidationState($v.file)"
              aria-describedby="image-file-help-block"
              @input="onFileUpload($event)"
            >
              <template #invalid>
                <b-form-invalid-feedback role="alert">
                  {{ $t('global.form.required') }}
                </b-form-invalid-feedback>
              </template>
            </form-file>
          </b-form-group>
        </template>

        <!-- TFTP Server Upload -->
        <template v-if="tftpServer">
          <b-form-group
            :label="$t('pageFirmware.form.updateFirmware.fileAddress')"
            label-for="tftp-address"
          >
            <b-form-input
              id="tftp-address"
              v-model="tftpFileAddress"
              type="text"
              :state="getValidationState($v.tftpFileAddress)"
              :disabled="isPageDisabled"
              @input="$v.tftpFileAddress.$touch()"
            />
            <b-form-invalid-feedback role="alert">
              <template v-if="!$v.tftpFileAddress.required">
                {{ $t('global.form.fieldRequired') }}
              </template>
            </b-form-invalid-feedback>
          </b-form-group>
        </template>
        <b-btn
          data-test-id="firmware-button-startUpdate"
          type="submit"
          variant="primary"
          :disabled="isPageDisabled"
        >
          {{ $t('pageFirmware.form.updateFirmware.startUpdate') }}
        </b-btn>
      </b-form>

      <!-- Progress Bar -->
      <b-progress
        v-if="showProgressBar"
        :max="100"
        animated
        class="mt-3"
        show-value
      >
        <b-progress-bar :value="progress" :variant="progressVariant">
          {{ progress }}%
        </b-progress-bar>
      </b-progress>
    </div>

    <!-- Modals -->
    <modal-update-firmware @ok="updateFirmware" />
  </div>
</template>

<script>
import { requiredIf } from 'vuelidate/lib/validators';
import BVToastMixin from '@/components/Mixins/BVToastMixin';
import LoadingBarMixin, { loading } from '@/components/Mixins/LoadingBarMixin';
import VuelidateMixin from '@/components/Mixins/VuelidateMixin.js';
import InfoTooltip from '@/components/Global/InfoTooltip';
import FormFile from '@/components/Global/FormFile';
import ModalUpdateFirmware from './FirmwareModalUpdateFirmware';

export default {
  name: 'FormUpdate',
  components: {
    InfoTooltip,
    FormFile,
    ModalUpdateFirmware,
  },
  mixins: [BVToastMixin, LoadingBarMixin, VuelidateMixin],
  props: {
    isPageDisabled: {
      required: true,
      type: Boolean,
      default: false,
    },
  },
  data() {
    return {
      loading,
      isWorkstationSelected: true,
      file: null,
      tftpFileAddress: null,
      isServerPowerOffRequired:
        process.env.VUE_APP_SERVER_OFF_REQUIRED === 'true',
      tftpServer: process.env.VUE_APP_TFTP_SERVER === 'true',
      progress: 0, // Track progress percentage
      showProgressBar: false, // Show progress bar by default
      toastRef: null,
    };
  },
  computed: {
    bmcPowerState() {
      return this.$store.getters['bmc/bmc']?.powerState;
    },
    bootProgress() {
      return this.$store.getters['global/bootProgress'];
    },
    isTftpUploadAvailable() {
      return this.$store.getters['firmware/isTftpUploadAvailable'];
    },
  },
  watch: {
    isWorkstationSelected: function () {
      this.$v.$reset();
      this.file = null;
      this.tftpFileAddress = null;
    },
    loading: function (value) {
      this.$emit('loadingStatus', value);
    },
  },
  validations() {
    return {
      file: {
        required: requiredIf(function () {
          return this.isWorkstationSelected;
        }),
      },
      tftpFileAddress: {
        required: requiredIf(function () {
          return !this.isWorkstationSelected;
        }),
      },
    };
  },
  created() {
    this.$store.dispatch('bmc/getBmcInfo');
    this.$store.dispatch('firmware/getUpdateServiceSettings');
  },
  methods: {
    /**
     * Generic method to update progress bar and show toast notifications
     * @param {Object} options
     * @param {number} options.percent - Progress percentage
     * @param {string} options.title - Toast title key
     * @param {string} options.message - Toast message key
     * @param {boolean} [options.isComplete=false] - Whether this is the final update
     * @param {boolean} [options.isError=false] - Whether this is an error state
     * @param {boolean} [options.refreshAction=false] - Whether to include refresh action in toast
     * @param {Function} [options.onComplete] - Callback to run on completion
     * @param {Function} [options.getProgress] - Optional function to fetch dynamic progress
     */
    updateProgress({
      percent,
      title,
      message,
      isComplete = false,
      isError = false,
      refreshAction = false,
      onComplete,
      getProgress,
    }) {
      if (!this.loading) {
        this.startLoader();
      }
      this.showProgressBar = true;

      // Set progress bar color
      this.progressVariant = isError ? 'danger' : 'success';

      // Use dynamic progress if getProgress is provided, else use provided percent
      this.progress = getProgress ? getProgress() : percent;

      if (isError) {
        // Show error toast and stop progress
        this.errorToast(this.$t(message), {
          title: this.$t(title),
          timestamp: true,
        });
        this.endLoader();
        //this.showProgressBar = false;
        //this.progress = 0;
        if (onComplete) onComplete();
        return;
      }

      // Show info toast for normal progress
      this.infoToast(this.$t(message, { progress: this.progress }), {
        title: this.$t(title),
        timestamp: true,
        ...(refreshAction && { refreshAction: true }),
      });

      if (isComplete) {
        this.endLoader();
        //this.showProgressBar = false;
        //this.progress = 0;
        if (onComplete) onComplete();
      }
    },
    createOrUpdateProgressToast({
      percent = 0,
      title = '',
      message = '',
      variant = 'info',
      hideAfter = false,
      height = '10px',
      onComplete = null,
    } = {}) {
      const h = this.$createElement;
      const content = h('div', [
        h('div', { class: 'mb-2' }, `${message} (${percent}%)`),
        h('b-progress', {
          props: {
            value: percent,
            max: 100,
            animated: true,
            height: height,
            variant: variant,
          },
        }),
      ]);

      // Hide previous toast if ref exists
      if (this.toastRef !== null) {
        this.$bvToast.hide(this.toastRef);
      }

      this.toastRef = this.$bvToast.toast([content], {
        title,
        toaster: 'b-toaster-top-right',
        solid: true,
        autoHide: hideAfter,
        noCloseButton: true,
        variant,
        appendToast: false,
      });

      if (hideAfter && typeof onComplete === 'function') {
        setTimeout(() => {
          this.$bvToast.hide(this.toastRef);
          onComplete();
        }, 3000);
      }
    },
    simulateFirmwareUpdate() {
      this.updateProgress({
        percent: 25,
        title: 'pageFirmware.toast.updateFirmware.step1',
        message: 'pageFirmware.toast.updateFirmware.step1Message',
      });

      setTimeout(() => {
        this.updateProgress({
          percent: 50,
          title: 'pageFirmware.toast.updateFirmware.step2',
          message: 'pageFirmware.toast.updateFirmware.step2Message',
        });
      }, 2000);

      setTimeout(() => {
        this.updateProgress({
          percent: 75,
          title: 'pageFirmware.toast.updateFirmware.step3',
          message: 'pageFirmware.toast.updateFirmware.step3Message',
          isError: true,
        });
      }, 3000);

      setTimeout(() => {
        this.updateProgress({
          percent: 100,
          title: 'pageFirmware.toast.updateFirmware.step4',
          message: 'pageFirmware.toast.updateFirmware.step4Message',
          isComplete: true,
          refreshAction: true,
        });
      }, 4000);
    },
    /*simulateFirmwareUpdate() {
      this.startLoader();
      this.progress = 0;
      this.showProgressBar = false; // Hide static bar, use toast

      const stages = [
        {
          percent: 25,
          title: this.$t('pageFirmware.toast.updateFirmware.step1'),
          message: this.$t('pageFirmware.toast.updateFirmware.step1Message')
        },
        {
          percent: 50,
          title: this.$t('pageFirmware.toast.updateFirmware.step2'),
          message: this.$t('pageFirmware.toast.updateFirmware.step2Message')
        },
        {
          percent: 75,
          title: this.$t('pageFirmware.toast.updateFirmware.step3'),
          message: this.$t('pageFirmware.toast.updateFirmware.step3Message')
        },
        {
          percent: 100,
          title: this.$t('pageFirmware.toast.updateFirmware.step4'),
          message: this.$t('pageFirmware.toast.updateFirmware.step4Message')
        }
      ];

      let currentStage = 0;
      let toastRef = null;

      const updateToast = () => {
        const stage = stages[currentStage];
        this.progress = stage.percent;

        const h = this.$createElement;
        const content = h('div', [
          h('div', { class: 'mb-2' }, `${stage.message} (${stage.percent}%)`),
          h('b-progress', {
            props: {
              value: stage.percent,
              max: 100,
              animated: true,
              height: '10px',
              variant: 'success'
            }
          })
        ]);

        if (toastRef === null) {
          toastRef = this.$bvToast.toast([content], {
            title: stage.title,
            toaster: 'b-toaster-top-right',
            solid: true,
            autoHide: false,
            noCloseButton: true,
            variant: 'info',
            appendToast: false
          });
        } else {
          this.$bvToast.hide(toastRef);
          toastRef = this.$bvToast.toast([content], {
            title: stage.title,
            toaster: 'b-toaster-top-right',
            solid: true,
            autoHide: false,
            noCloseButton: true,
            variant: 'info',
            appendToast: false
          });
        }

        // this.$bvToast.toast([content], {
        //   $bvToast.hide(toastId),
        //   id: toastId,
        //   title: stage.title,
        //   toaster: 'b-toaster-top-right',
        //   solid: true,
        //   autoHide: false,
        //   noCloseButton: true,
        //   variant: 'info',
        //   appendToast: true
        // });

        currentStage++;
        if (currentStage < stages.length) {
          setTimeout(updateToast, 3000);
        } else {
          setTimeout(() => {
            this.$bvToast.hide(toastRef);
            this.endLoader();
            this.progress = 0;
          }, 3000);
        }
      };

      updateToast();
    },*/
    updateFirmware() {
      this.startLoader();
      this.$emit('loadingStatus', this.loading);

      // Step 1 - Upload
      const uploadFirmware = () => {
        console.log('uploadFirmware++');
        this.updateProgress({
          percent: 25,
          title: 'pageFirmware.toast.updateFirmware.step1',
          message: 'pageFirmware.toast.updateFirmware.step1Message',
        });
        console.log('uploadFirmware--');
        if (this.isWorkstationSelected) {
          this.dispatchWorkstationUpload(activateFirmware);
        } else {
          this.dispatchTftpUpload(activateFirmware);
        }
      };
      console.log('Going to activate...');
      // Step 2 - Activation
      const activateFirmware = async (data) => {
        const taskLink = data['@odata.id'];
        console.log('activateFirmware++');

        const currentTask = async () => {
          return await this.$store.dispatch('global/getCurrentTask', taskLink);
        };

        const currentTaskProgress = (checkCounter = 0, data) => {
          checkCounter++;

          // This counter goes up by 1 every time this function runs
          // If the function successfully goes to last toast, it won't run anymore
          // if this function runs more than 36 times, it won't run anymore
          if (checkCounter > 36) {
            this.endLoader();
            return this.errorToast(
              this.$t('pageFirmware.toast.errorActivation')
            );
          }

          Promise.all([currentTask(data)]).then((res) => {
            // Check to see if activation was aborted
            const activationAborted = res[0].Messages.filter((message) =>
              message.MessageId.endsWith('TaskAborted')
            )[0];

            if (activationAborted) {
              if (activationAborted?.Oem?.OpenBMC?.AbortReason) {
                const message = activationAborted?.Oem?.OpenBMC?.AbortReason?.split(
                  '.'
                ).pop();
                if (message === 'ExpiredAccessKey')
                  return this.errorToast(
                    this.$t('pageFirmware.toast.expiredAccessKeyError')
                  );
              } else {
                return this.errorToast(
                  this.$t('pageFirmware.toast.errorActivation')
                );
              }
            }

            // res[0].error indicates that the activation was completed and removed
            // because of BMC starting reboot
            if (res[0].PercentComplete == 100 || res[0].error) {
              bmcReboot();
            } else {
              setTimeout(() => {
                currentTaskProgress(checkCounter, data);
              }, 5000); // If the percent complete isn't 100 or complete, run again in 5 sec
            }
          });
        };

        if (taskLink) {
          this.updateProgress({
            percent: 50,
            title: 'pageFirmware.toast.updateFirmware.step2',
            message: 'pageFirmware.toast.updateFirmware.step2Message',
          });
          console.log('activateFirmware--');
          currentTaskProgress(0, taskLink);
        } else {
          this.endLoader();
          return this.errorToast(this.$t('pageFirmware.toast.errorActivation'));
        }
      };
      console.log('Going to BMC reboot');
      // Step 3 - BMC Reboot
      const bmcReboot = async () => {
        this.updateProgress({
          percent: 75,
          title: 'pageFirmware.toast.updateFirmware.step3',
          message: 'pageFirmware.toast.updateFirmware.step3Message',
        });

        const rebootProgress = async () => {
          setTimeout(async () => {
            await this.$store
              .dispatch('bmc/getBmcInfo')
              .then(() => {
                if (this.bmcPowerState === 'On') {
                  activationComplete();
                } else {
                  rebootProgress();
                }
              })
              .catch(({ message }) => {
                this.endLoader();
                this.errorToast(message);
              });
          }, 180000); // 3 minutes
        };
        rebootProgress();
      };

      // Step 4 - Activation complete
      const activationComplete = () => {
        this.endLoader();
        this.updateProgress({
          percent: 100,
          title: 'pageFirmware.toast.updateFirmware.step4',
          message: 'pageFirmware.toast.updateFirmware.step4Message',
          isComplete: true,
          refreshAction: true,
        });
      };

      uploadFirmware(); // This must be here to run the entire function
    },
    dispatchWorkstationUpload(activateFirmware) {
      // Handle case where file might be null
      // if (!this.file) {
      //   // Optionally, add a warning or proceed with a default behavior
      //   console.warn('No file selected, proceeding with upload');
      // }
      this.$store
        .dispatch('firmware/uploadFirmware', this.file)
        .then(async ({ data }) => {
          activateFirmware(data);
        })
        .catch(({ message }) => {
          this.endLoader();
          this.errorToast(message);
        });
    },
    dispatchTftpUpload(activateFirmware) {
      this.$store
        .dispatch('firmware/uploadFirmwareTFTP', this.tftpFileAddress)
        .then(({ data }) => {
          activateFirmware(data);
        })
        .catch(({ message }) => {
          this.endLoader();
          this.errorToast(message);
        });
    },
    onSubmitUpload() {
      this.$v.$touch();
      if (this.$v.$invalid) return;
      // Skip validation for file field when isWorkstationSelected is true
      if (!this.isWorkstationSelected) {
        this.$v.tftpFileAddress.$touch();
        if (this.$v.tftpFileAddress.$invalid) return;
      }
      this.$bvModal.show('modal-update-firmware');
    },
    onFileUpload(file) {
      this.file = file;
      this.$v.file.$touch();
    },
  },
};
</script>

<!-- <style scoped>
/* Optional: Customize progress bar appearance */
.progress {
  height: 30px;
  font-size: 14px;
}

/* Style for toast progress bar */
:deep(.toast-body .progress) {
  height: 10px;
  /* Match toast progress bar height */
  font-size: 12px;
}
</style>̦ -->
