<script>
import { WORKLOAD_TYPES } from '@/config/types';
import UnitInput from '@/components/form/UnitInput';
import LabeledInput from '@/components/form/LabeledInput';
import RadioGroup from '@/components/form/RadioGroup';

export default {
  components: {
    UnitInput, LabeledInput, RadioGroup
  },
  props:      {
    value: {
      type:    Object,
      default: () => {
        return {};
      }
    },
    mode: {
      type:     String,
      required: true,
    },
    type: {
      type:    String,
      default: WORKLOAD_TYPES.JOB
    }
  },
  data() {
    const {
      failedJobsHistoryLimit, successfulJobsHistoryLimit, suspend, schedule
    } = this.value;

    if (this.type === WORKLOAD_TYPES.CRON_JOB) {
      if (!this.value.jobTemplate) {
        this.$set(this.value, 'jobTemplate', { spec: {} });
      }
      const {
        completions, parallelism, backOffLimit, activeDeadlineSeconds
      } = this.value.jobTemplate.spec;

      return {
        completions, parallelism, backOffLimit, activeDeadlineSeconds, failedJobsHistoryLimit, successfulJobsHistoryLimit, suspend, schedule
      };
    } else {
      const {
        completions, parallelism, backOffLimit, activeDeadlineSeconds
      } = this.value;

      return {
        completions, parallelism, backOffLimit, activeDeadlineSeconds, failedJobsHistoryLimit, successfulJobsHistoryLimit, suspend, schedule
      };
    }
  },
  computed: {
    isCronJob() {
      return this.type === WORKLOAD_TYPES.CRON_JOB;
    }
  },
  methods: {
    update() {
      if (this.type === WORKLOAD_TYPES.JOB) {
        const spec = {
          ...this.value,
          suspend:                    this.suspend,
          schedule:                   this.schedule,
          completions:           this.completions,
          parallelism:           this.parallelism,
          backOffLimit:          this.backOffLimit,
          activeDeadlineSeconds: this.activeDeadlineSeconds,

        };

        this.$emit('input', spec);
      } else {
        const spec = {
          ...this.value,
          failedJobsHistoryLimit:     this.failedJobsHistoryLimit,
          successfulJobsHistoryLimit: this.successfulJobsHistoryLimit,
          suspend:                    this.suspend,
          schedule:                   this.schedule,
          jobTemplate:                {
            ...this.value.jobTemplate,
            completions:           this.completions,
            parallelism:           this.parallelism,
            backOffLimit:          this.backOffLimit,
            activeDeadlineSeconds: this.activeDeadlineSeconds
          }
        };

        this.$emit('input', spec);
      }
    }
  }
};
</script>

<template>
  <form class="pl-10" @input="update">
    <div class="row">
      <div class="col span-6">
        <UnitInput v-model="completions" :mode="mode" :suffix="completions===1 ? 'Time' : 'Times'">
          <template v-slot:label>
            <label :style="{'color':'var(--input-label)'}">
              <t k="workload.jobConfig.completions.label" />
              <i v-tooltip="t('workload.jobConfig.completions.detail')" class="icon icon-info" style="font-size: 14px" />
            </label>
          </template>
        </UnitInput>
      </div>
      <div class="col span-6">
        <UnitInput v-model="parallelism" :mode="mode" class="col span-6" :suffix="parallelism===1 ? 'Time' : 'Times'">
          <template v-slot:label>
            <label :style="{'color':'var(--input-label)'}">
              <t k="workload.jobConfig.parallelism.label" />
              <i v-tooltip="t('workload.jobConfig.parallelism.detail')" class="icon icon-info" style="font-size: 14px" />
            </label>
          </template>
        </UnitInput>
      </div>
    </div>
    <div class="row">
      <div class="col span-6">
        <UnitInput v-model="backOffLimit" :mode="mode" :suffix="backOffLimit===1 ? 'Time' : 'Times'">
          <template v-slot:label>
            <label :style="{'color':'var(--input-label)'}">
              <t k="workload.jobConfig.backoffLimit.label" />
              <i v-tooltip="t('workload.jobConfig.backoffLimit.detail')" class="icon icon-info" style="font-size: 14px" />
            </label>
          </template>
        </UnitInput>
      </div>
      <div class="col span-6">
        <UnitInput v-model="activeDeadlineSeconds" :mode="mode" :suffix="activeDeadlineSeconds===1 ? 'Second' : 'Seconds'">
          <template v-slot:label>
            <label :style="{'color':'var(--input-label)'}">
              <t k="workload.jobConfig.activeDeadlineSeconds.label" />
              <i v-tooltip="t('workload.jobConfig.activeDeadlineSeconds.detail')" class="icon icon-info" style="font-size: 14px" />
            </label>
          </template>
        </UnitInput>
      </div>
    </div>
    <template v-if="isCronJob">
      <div class="row">
        <div class="col span-6">
          <LabeledInput v-model="successfulJobsHistoryLimit" :mode="mode">
            <template v-slot:label>
              <label :style="{'color':'var(--input-label)'}">
                Successful Job History Limit
                <i v-tooltip="'The number of successful finished jobs to retain.'" class="icon icon-info" style="font-size: 14px" />
              </label>
            </template>
          </LabeledInput>
        </div>
        <div class="col span-6">
          <LabeledInput v-model="failedJobsHistoryLimit" :mode="mode">
            <template v-slot:label>
              <label :style="{'color':'var(--input-label)'}">
                Failed Job History Limit
                <i v-tooltip="'The number of failed finished jobs to retain.'" class="icon icon-info" style="font-size: 14px" />
              </label>
            </template>
          </LabeledInput>
        </div>
      </div>
      <span>Suspend</span>
      <RadioGroup v-model="suspend" row :options="[true, false]" :labels="['Yes', 'No']" />
    </template>
  </form>
</template>
