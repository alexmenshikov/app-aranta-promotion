<script setup>
import {ref, watch} from "vue";
import {
  Col as ACol,
  Form as AForm,
  FormItem as AFormItem,
  Row as ARow,
  Select as ASelect,
  Input as AInput,
  Card as ACard,
  Tag as ATag,
  Switch as ASwitch,
  Skeleton as ASkeleton,
  ConfigProvider as AConfigProvider,
} from "ant-design-vue";
import {
  CheckOutlined,
  CloseOutlined
} from '@ant-design/icons-vue';
import ruRu from "ant-design-vue/es/locale/ru_RU";
import dayjs from "dayjs";

const props = defineProps({
  campaign: {
    type: Object,
    required: true,
  },
  loading: {
    type: Boolean,
    default: false,
  },
});

const emit = defineEmits([
  "updateEnabled",
  "updateImpressions",
  "updateInterval",
  "updateUpdated",
]);

function formatNumberWithSpacesAndSymbol(number) {
  const numStr = String(number).split('.');

  const integerPart = numStr[0];
  const decimalPart = numStr[1];

  const formattedIntegerPart = integerPart.replace(/\B(?=(\d{3})+(?!\d))/g, " ");

  return decimalPart ? formattedIntegerPart + '.' + decimalPart + ' ₽' : formattedIntegerPart + ' ₽';
}

function getTagColorStatus(value) {
  const tagColors = {
    "-1": "#FF4D4F", // кампания в процессе удаления
    4: "#52C41A", // готова к запуску
    7: "#BFBFBF", // кампания завершена
    8: "#FA8C16", // отказался
    9: "#1890FF", // идут показы
    11: "#FAAD14", // кампания на паузе
  };
  return tagColors[value]
}

const intervalOptions = [
  {
    label: "15 минут",
    value: 15,
  },
  {
    label: "20 минут",
    value: 20,
  },
  {
    label: "30 минут",
    value: 30,
  }
];

// Локальные переменные для хранения состояния
const enabled = ref(props.campaign.enabled);
const impressions = ref(props.campaign.impressions);
const interval = ref(props.campaign.interval);

function checkingAndDisplayingValue(label, value) {
  return `${label} ${value !== null ? value : "<u>нет значения</u>"}`;
}

function emitEnabledChange(checked) {
  emit("updateEnabled", checked);
}

const emitInputChange = (value) => {
  emit("updateImpressions", Number(value));
};

const emitIntervalChange = (value) => {
  emit("updateInterval", value);
};
</script>

<template>
  <a-config-provider :locale="ruRu">
    <a-card :title="campaign.name" class="card">
      <template #extra>
        <a-switch :disabled="loading" v-model:checked="enabled" @change="emitEnabledChange">
          <template #checkedChildren><check-outlined /></template>
          <template #unCheckedChildren><close-outlined /></template>
        </a-switch>
      </template>

      <a-skeleton v-if="loading" active :title="false" :paragraph="{ rows: 4, width: '100%' }" />

      <div v-else class="content">
        <div class="info">
          <span class="grey">{{ campaign.type.convert }}</span>

          <span class="grey">ID {{ campaign.advertId }}</span>

          <span v-html="checkingAndDisplayingValue('Текущая ставка', formatNumberWithSpacesAndSymbol(campaign.cpm))" class="grey"></span>

          <span class="grey">Бюджет {{ formatNumberWithSpacesAndSymbol(campaign.budget) }}</span>

          <span v-html="checkingAndDisplayingValue('CTR', campaign.ctr)" class="grey"></span>

          <span v-html="checkingAndDisplayingValue('Показы', campaign.views )" class="grey"></span>

          <span><a-tag :bordered="false" :color="getTagColorStatus(campaign.status.original)">{{ campaign.status.convert }}</a-tag></span>
        </div>

        <a-form ref="form" layout="vertical">
          <a-row :gutter="24">
            <a-col :span="4">
              <a-form-item label="Интервал" name="interval">
                <a-select
                  v-model:value="interval"
                  :options="intervalOptions"
                  @select="emitIntervalChange"
                  :disabled="enabled"
                />
              </a-form-item>
            </a-col>
            <a-col :span="4">
              <a-form-item label="Кол-во показов в сутки" name="numberOfImpressionsPerDay">
                <a-input
                  v-model:value.number="impressions"
                  @input="emitInputChange($event.target.value)"
                  placeholder="Кол-во показов"
                  :disabled="enabled"
                />
              </a-form-item>
            </a-col>
          </a-row>
        </a-form>

        <div>
          <span>Обновлено {{ dayjs(campaign.updated).format("DD.MM.YYYY HH:mm") }}</span>
        </div>
      </div>
    </a-card>
  </a-config-provider>
</template>

<style scoped>
.info {
  display: flex;
  column-gap: 15px;
  margin-bottom: 24px;
}

.grey {
  color: rgba(0, 0, 0, 0.5);
}

.card {
  background-color: rgba(0, 0, 0, 0.01);
}
</style>