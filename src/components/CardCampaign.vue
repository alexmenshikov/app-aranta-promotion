<script setup>
import { ref } from "vue";
import {
  Card as ACard,
  Tag as ATag,
  Switch as ASwitch,
} from "ant-design-vue";
import {
  CheckOutlined,
  CloseOutlined
} from '@ant-design/icons-vue';

const props = defineProps({
  campaign: {
    type: Object,
    required: true,
  },
});

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

const state = ref(false);
</script>

<template>
  <a-card :title="campaign.name">
    <template #extra>
      <a-switch v-model:checked="state">
        <template #checkedChildren><check-outlined /></template>
        <template #unCheckedChildren><close-outlined /></template>
      </a-switch>
    </template>

    <div class="info">
      <span class="grey">{{ campaign.type.convert }}</span>
      <span class="grey">ID {{ campaign.advertId }}</span>
      <span class="grey">Текущая ставка {{ formatNumberWithSpacesAndSymbol(campaign.cpm) }}</span>
      <span class="grey">Бюджет {{ formatNumberWithSpacesAndSymbol(campaign.budget) }}</span>
      <span class="grey">CTR {{ campaign.ctr }}</span>
      <span class="grey">Просмотров {{ campaign.views }}</span>
      <span><a-tag :bordered="false" :color="getTagColorStatus(campaign.status.original)">{{ campaign.status.convert }}</a-tag></span>
    </div>
  </a-card>
</template>

<style scoped>
.info {
  display: flex;
  column-gap: 15px;
}

.grey {
  color: rgba(0, 0, 0, 0.5);
}
</style>