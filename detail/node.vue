<script>
import { capitalize, words } from 'lodash';
import ConsumptionGauge from '@/components/ConsumptionGauge';
import DetailTop from '@/components/DetailTop';
import HStack from '@/components/Layout/Stack/HStack';
import VStack from '@/components/Layout/Stack/VStack';
import Alert from '@/components/Alert';
import SortableTable from '@/components/SortableTable';
import { allSettled } from '@/utils/promise';
import Tab from '@/components/Tabbed/Tab';
import BadgeState from '@/components/BadgeState';
import {
  ADDRESS,
  EFFECT,
  IMAGE_SIZE,
  KEY,
  LAST_HEARTBEAT_TIME,
  MESSAGE,
  REASON,
  SIMPLE_NAME,
  SIMPLE_TYPE,
  STATUS,
  VALUE,
} from '@/config/table-headers';
import ResourceTabs from '@/components/form/ResourceTabs';
import Poller from '@/utils/poller';
import { METRIC, POD } from '@/config/types';
import createEditView from '@/mixins/create-edit-view';
import { formatSi, exponentNeeded, UNITS } from '@/utils/units';
import CopyToClipboardText from '@/components/CopyToClipboardText';

const METRICS_POLL_RATE_MS = 30000;
const MAX_FAILURES = 2;

export default {
  name: 'DetailNode',

  components: {
    Alert,
    BadgeState,
    ConsumptionGauge,
    CopyToClipboardText,
    DetailTop,
    HStack,
    VStack,
    ResourceTabs,
    Tab,
    SortableTable
  },

  mixins: [createEditView],

  props: {
    value: {
      type:     Object,
      required: true,
    },
  },

  data() {
    return {
      metricPoller:           new Poller(this.loadMetrics, METRICS_POLL_RATE_MS, MAX_FAILURES),
      metrics:                { cpu: 0, memory: 0 },
      podsRequested:          0,
      conditionsTableHeaders: [
        SIMPLE_TYPE,
        STATUS,
        LAST_HEARTBEAT_TIME,
        REASON,
        MESSAGE
      ],
      infoTableHeaders: [
        {
          ...KEY,
          label: '',
          width: 200
        },
        {
          ...VALUE,
          label: ''
        }
      ],
      addressTableHeaders: [
        SIMPLE_TYPE,
        ADDRESS
      ],
      imageTableHeaders: [
        { ...SIMPLE_NAME, width: 900 },
        IMAGE_SIZE
      ],
      taintTableHeaders: [
        KEY,
        VALUE,
        EFFECT
      ]
    };
  },

  computed: {
    memoryUnits() {
      const exponent = exponentNeeded(this.value.ramCapacity, 1024);

      return `${ UNITS[exponent] }iB`;
    },

    pidPressureStatus() {
      return this.mapToStatus(this.value.isPidPressureOk);
    },

    diskPressureStatus() {
      return this.mapToStatus(this.value.isDiskPressureOk);
    },

    memoryPressureStatus() {
      return this.mapToStatus(this.value.isMemoryPressureOk);
    },

    kubeletStatus() {
      return this.mapToStatus(this.value.isKubeletOk);
    },

    conditionsTableRows() {
      return this.value.status.conditions;
    },

    infoTableRows() {
      return Object.keys(this.value.status.nodeInfo)
        .map(key => ({
          key:   capitalize(words(key).join(' ')),
          value: this.value.status.nodeInfo[key]
        }));
    },

    addressTableRows() {
      return this.value.status.addresses;
    },

    imageTableRows() {
      return this.value.status.images.map(image => ({
        // data in pos 1 has version/branch, data in pos 0 has sha, but too long
        // sometimes it does not include 1
        name:      image.names[1] || image.names[0],
        sizeBytes: image.sizeBytes
      }));
    },

    taintTableRows() {
      return this.value.spec.taints || [];
    },

    detailTopColumns() {
      return [
        {
          title:   this.t('node.detail.detailTop.ipAddress'),
          name:    'ip-address',
          icon:    '#icon-ip'
        },
        {
          title:   this.t('node.detail.detailTop.version'),
          content:  this.value.version,
          icon:    '#icon-version',
        },
        {
          title:   this.t('node.detail.detailTop.os'),
          content:  this.value.status.nodeInfo.osImage,
          icon:    '#icon-sys'
        },
        {
          title:   this.t('node.detail.detailTop.containerRuntime'),
          name:    'container-runtime',
          icon:    '#icon-runtime'
        },
      ];
    }
  },

  mounted() {
    this.metricPoller.start();
  },

  beforeDestroy() {
    this.metricPoller.stop();
  },

  methods: {
    memoryFormatter(value) {
      const formatOptions = {
        addSuffix:   false,
        increment:   1024,
        minExponent: 3
      };

      return formatSi(value, formatOptions);
    },

    mapToStatus(isOk) {
      return isOk
        ? 'success'
        : 'error';
    },
    async loadMetrics() {
      const schema = this.$store.getters['cluster/schemaFor'](METRIC.NODE);

      if (schema) {
        const ret = await allSettled({
          metrics: this.$store.dispatch('cluster/find', {
            type: METRIC.NODE,
            id:   this.value.id,
            opt:  { force: true }
          }),
          pods:    this.$store.dispatch('management/findAll', {
            type: POD,
            opt:  { force: true }
          })
        });

        const nodeName = this.value.id;
        const pods = ret.pods?.filter((podItem) => {
          return podItem.spec.nodeName === nodeName;
        }) || [];

        this.podsRequested = pods.length;

        this.$forceUpdate();
      }
    }
  }
};
</script>

<template>
  <VStack class="node">
    <DetailTop :columns="detailTopColumns">
      <template v-slot:ip-address>
        <CopyToClipboardText :text="value.internalIp" :use-icon="true" />
      </template>
      <template v-slot:state>
        <BadgeState v-if="value.showDetailStateBadge" :value="value" />
      </template>
      <template v-slot:container-runtime>
        <span><span v-if="value.containerRuntimeIcon" class="icon" :class="value.containerRuntimeIcon" /> {{ value.containerRuntimeVersion }}</span>
      </template>
    </DetailTop>
    <HStack class="glance" :show-dividers="true">
      <VStack class="alerts card-box-shadow mr-10" :show-dividers="true" vertical-align="space-evenly">
        <Alert :status="pidPressureStatus" :message="t('node.detail.glance.pidPressure')" />
        <Alert :status="diskPressureStatus" :message="t('node.detail.glance.diskPressure')" />
        <Alert :status="memoryPressureStatus" :message="t('node.detail.glance.memoryPressure')" />
        <Alert :status="kubeletStatus" :message="t('node.detail.glance.kubelet')" />
      </VStack>
      <HStack class="cluster card-box-shadow" horizontal-align="space-evenly">
        <ConsumptionGauge :resource-name="t('node.detail.glance.consumptionGauge.cpu')" :capacity="value.cpuCapacity" units="核" :used="value.cpuUsage" />
        <ConsumptionGauge :resource-name="t('node.detail.glance.consumptionGauge.memory')" :capacity="value.ramCapacity" :used="value.ramUsage" :units="memoryUnits" :number-formatter="memoryFormatter" />
        <ConsumptionGauge :resource-name="t('node.detail.glance.consumptionGauge.pods')" :capacity="value.podCapacity" :used="podsRequested" />
      </HStack>
    </HStack>
    <ResourceTabs v-model="value" :mode="mode">
      <template v-slot:before>
        <Tab name="conditions" :label="t('node.detail.tab.conditions')">
          <SortableTable
            key-field="_key"
            class="pl-10"
            :headers="conditionsTableHeaders"
            :rows="conditionsTableRows"
            :row-actions="false"
            :table-actions="false"
            :search="false"
            :use-events-table="true"
          />
        </Tab>
        <Tab name="info" :label="t('node.detail.tab.info')">
          <SortableTable
            key-field="_key"
            class="pl-10"
            :headers="infoTableHeaders"
            :rows="infoTableRows"
            :row-actions="false"
            :table-actions="false"
            :show-headers="false"
            :search="false"
          />
        </Tab>
        <Tab name="address" :label="t('node.detail.tab.address')">
          <SortableTable
            key-field="_key"
            class="pl-10"
            :headers="addressTableHeaders"
            :rows="addressTableRows"
            :row-actions="false"
            :table-actions="false"
            :search="false"
          />
        </Tab>
        <Tab name="images" :label="t('node.detail.tab.images')">
          <SortableTable
            key-field="_key"
            class="pl-10"
            :headers="imageTableHeaders"
            :rows="imageTableRows"
            :row-actions="false"
            :table-actions="false"
          />
        </Tab>
        <Tab name="taints" :label="t('node.detail.tab.taints')">
          <SortableTable
            key-field="_key"
            class="pl-10"
            :headers="taintTableHeaders"
            :rows="taintTableRows"
            :row-actions="false"
            :table-actions="false"
            :search="false"
          />
        </Tab>
      </template>
    </ResourceTabs>
  </VStack>
</template>

<style lang="scss" scoped>
.cluster {
  flex: 1;
}

$divider-spacing: 20px;

.glance {
  margin-top: 20px;

  & > * {
    padding: 0 $divider-spacing;
  }
}

.alerts {
  width: 20%;
  & > * {
    flex: 1;
  }
}
</style>
