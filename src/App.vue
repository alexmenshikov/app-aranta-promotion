<script setup>
import axios from "axios";
import CardCampaign from "./components/CardCampaign.vue";
import {computed, onMounted, ref, watch} from "vue";
import {
  Col as ACol,
  ConfigProvider as AConfigProvider,
  Form as AForm,
  FormItem as AFormItem,
  Row as ARow,
  Select as ASelect,
  Button as AButton,
  message,
} from "ant-design-vue";
import {
  ReloadOutlined
} from "@ant-design/icons-vue"
import ruRu from "ant-design-vue/es/locale/ru_RU";
import dayjs from "dayjs";
import ru from "dayjs/locale/ru";
import utc from "dayjs/plugin/utc";

dayjs.locale(ru);
dayjs.extend(utc);

const companyArray = [
  {
    id: 1,
    name: "Ne Vi (WB)",
    apiToken: "eyJhbGciOiJFUzI1NiIsImtpZCI6IjIwMjQxMDE2djEiLCJ0eXAiOiJKV1QifQ.eyJlbnQiOjEsImV4cCI6MTc0NDkxOTQzNiwiaWQiOiIwMTkyOTk3NS0wODQzLTdhM2EtYTJiNS1kN2VjODUyZTBmNjYiLCJpaWQiOjE5NjI0NzM2LCJvaWQiOjQxMjc0NjcsInMiOjk2LCJzaWQiOiI4NGI5ZDZkMy0wMTEyLTQwYmYtOTE2Yi1lZWQxZDhmNzYwYTUiLCJ0IjpmYWxzZSwidWlkIjoxOTYyNDczNn0.b2vg9TaUFcMQekb-N0CzC0MiuTShlBX52HnWRV2RDFagKIvYIx_EBtrRRKF5RTKCC3eZSKOKZAUQAG6k6aVLbg",
  },
  {
    id: 2,
    name: "Sunflowers (WB)",
    apiToken: "eyJhbGciOiJFUzI1NiIsImtpZCI6IjIwMjQxMDE2djEiLCJ0eXAiOiJKV1QifQ.eyJlbnQiOjEsImV4cCI6MTc0NTY5MDE1OCwiaWQiOiIwMTkyYzc2NS01MGM2LTdkMjItODI0MC05YjFhODkzMzY1NGQiLCJpaWQiOjk2OTgyNDY4LCJvaWQiOjQwNzg0NjMsInMiOjk2LCJzaWQiOiIwZjM2NDYwMy1hODFjLTQzYWQtOTI5Yi0yYWYzMTlhYWU3M2MiLCJ0IjpmYWxzZSwidWlkIjo5Njk4MjQ2OH0.MR_oL3iLR20FPAeE5SgKoufYhPU6dkNAH6YqHm4SoK2VfEwFribBuj3dZOLVdolQgPcCppijESMcuo6KnXs8uA",
  },
];

const transformedCompanyOptions = computed(() => {
  return companyArray.map(company => ({
    label: company.name,
    value: JSON.stringify(company)
  }));
});

const companyOptions = ref([]);
const companySelected = ref(JSON.stringify(companyArray[0]));
const loading = ref(false);
const disabledUpdateCampaign = ref(false);

watch(disabledUpdateCampaign, (newValue) => {
  if (newValue) {
    setTimeout(function () {
      disabledUpdateCampaign.value = false;
    }, 180000);
  }
});

// watch(companySelected, (newValue, oldValue) => {
  // localStorage.setItem("company_selected_feedbacks", JSON.stringify(newValue));
  //
  // if (newValue !== oldValue) {
  //   handleStop();
  //
  //   initValues();
  // }
// });

companyOptions.value = transformedCompanyOptions.value;
const transformedCompanySelected = computed(() => JSON.parse(companySelected.value));

// const listCampaignsIds = ref({ ids: [], objectIds: [] });
const campaigns = ref([]);

function convertCampaignsType(value) {
  const campaignTypes = {
    8: "Автоматическая",
    9: "Аукцион"
  };

  return campaignTypes[value];
}

function convertCampaignsStatus(value) {
  const campaignStatus = {
    "-1": "кампания в процессе удаления",
    4: "готова к запуску",
    7: "кампания завершена",
    8: "отказался",
    9: "идут показы",
    11: "кампания на паузе"
  };

  return campaignStatus[value];
}

// Функция для получения бюджета
async function getCampaignBudget(id) {
  const loadingMessage = message.loading(`Получение бюджета у кампании ${id}`, 0);

  return axios
    .get(`https://advert-api.wildberries.ru/adv/v1/budget?id=${id}`, {
      headers: {
        Authorization: `${transformedCompanySelected.value.apiToken}`
      }
    })
    .then(response => response.data.total)
    .catch(error => {
      console.log("getCampaignBudget: ", error);
      loadingMessage();
      message.error(`Ошибка при загрузке бюджета у кампании ${id}`);
      return null;
    })
    .finally(() => {
      loadingMessage();
    });
}

async function updateCampaignBudgets(campaignsValue) {
  const campaignIds = campaignsValue.map(campaign => campaign.advertId);

  // Перебираем каждый элемент newIds и обновляем budget
  for (const id of campaignIds) {
    const budget = await getCampaignBudget(id);

    // Находим объект в campaigns с совпадающим advertId
    const campaign = campaigns.value.find(
      item => item.advertId === id
    );

    if (campaign) {
      campaign.budget = budget;
    }
  }
}

// Функция для получения CTR
async function updateCampaignCtr(campaignsValue) {
  const campaignIds = campaignsValue.map(campaign => ({ id: campaign.advertId }));
  const loadingMessage = message.loading('Получение CTR для кампаний', 0);
  loading.value = true;

  axios
    .post("https://advert-api.wildberries.ru/adv/v2/fullstats", campaignIds,{
      headers: {
        Authorization: `${transformedCompanySelected.value.apiToken}`
      }
    })
    .then(response => {
      for (const element of response.data) {
        // Находим объект в campaigns с совпадающим advertId
        const campaign = campaigns.value.find(
          item => item.advertId === element.advertId
        );

        if (campaign) {
          campaign.ctr = element.ctr;
          campaign.views = element.views;
        }
      }

      loading.value = false;
    })
    .catch(error => {
      console.log("getCampaignCtr: ", error);
      loading.value = false;
      loadingMessage();
      message.error("Ошибка при загрузке CTR");
    })
    .finally(() => {
    loadingMessage();
    });
}

// Функция для получения информации о компаниях
async function getInformationAboutCampaigns(campaignsValue) {
  const campaignIds = campaignsValue.map(campaign => campaign.advertId);
  const loadingMessage = message.loading('Загрузка информации о кампаниях', 0);

  try {
    const response = await axios.post("https://advert-api.wildberries.ru/adv/v1/promotion/adverts", campaignIds, {
        headers: {
          Authorization: `${transformedCompanySelected.value.apiToken}`,
        },
      }
    );

    response.data.forEach(newItem => {
      const campaign = campaigns.value.find(item => item.advertId === newItem.advertId);
      if (campaign) {
        campaign.name = newItem.name;
        campaign.type.original = newItem.type;
        campaign.type = {
          original: newItem.type,
          convert: convertCampaignsType(newItem.type)
        };
        campaign.status = {
          original: newItem.status,
          convert: convertCampaignsStatus(newItem.status)
        };
        campaign.cpm = newItem.unitedParams ? newItem.unitedParams[0].catalogCPM : newItem.autoParams.cpm;
        // campaign.budget = newItem.dailyBudget;
        campaign.updated = new Date().toISOString();
        // campaign.enabled = newItem.searchPluseState || newItem.autoParams.active.carousel;
        // campaign.impressions = newItem.unitedParams ? newItem.unitedParams[0].searchCPM : newItem.autoParams.nmCPM[0].cpm;
      }
    });
    // campaigns.value = response.data.map(item => ({
    //   name: item.name,
    //   advertId: item.advertId,
    //   type: {
    //     original: item.type,
    //     convert: convertCampaignsType(item.type),
    //   },
    //   status: {
    //     original: item.status,
    //     convert: convertCampaignsStatus(item.status),
    //   },
    //   cpm: item.type === 8 ? item.autoParams.nmCPM[0].cpm : item.unitedParams[0].searchCPM,
    //   budget: null,
    //   ctr: null,
    //   views: null,
    //   // updated: dayjs.utc().local().format("DD.MM.YYYY HH:mm"),
    //   updated: new Date().toISOString(),
    //   enabled: false,
    //   impressions: null,
    //   interval: {
    //     value: 15,
    //     repeat: 4
    //   },
    // }));
  } catch (error) {
    console.error("getInformationAboutCampaigns", error);
    loadingMessage();
    message.error("Ошибка при загрузке информации о кампаниях");
  } finally {
    loadingMessage();
  }
}

// Следим за изменениями в listCampaignsIds, если нужно автоматически вызывать вторую функцию
// watch(() => listCampaignsIds.value.ids, (newIds) => {
//     if (newIds.length > 0) {
//       // Вызов getInformationAboutCampaigns с заполненными данными
//       getInformationAboutCampaigns(newIds);
//       updateCampaignBudgets(newIds);
//     }
//   }
// );
//
// watch(() => listCampaignsIds.value.objectIds, (newIds) => {
//     if (newIds.length > 0) {
//       updateCampaignCtr(newIds);
//     }
//   }
// );

// Функция для получения кампаний
async function getListOfCampaigns() {
  const loadingMessage = message.loading('Загрузка кампаний', 0);
  loading.value = true;
  disabledUpdateCampaign.value = true;

  try {
    const response = await axios.get("https://advert-api.wildberries.ru/adv/v1/promotion/count", {
      headers: {
        Authorization: `${transformedCompanySelected.value.apiToken}`,
      },
      }
    );

    const advertListIds = response.data.adverts.flatMap((advert) =>
      advert.advert_list.map((item) => item.advertId)
    );

    campaigns.value = advertListIds.map(advertId => ({
      name: null,
      advertId: advertId,
      type: {
        original: null,
        convert: null,
      },
      status: {
        original: null,
        convert: null,
      },
      cpm: null,
      budget: null,
      ctr: null,
      views: null,
      updated: null,
      enabled: false,
      impressions: null,
      interval: 15,
    }));
    loading.value = false;
  } catch (error) {
    loadingMessage();
    message.error("Ошибка при загрузке кампаний");
    console.error("getListOfCampaigns", error);
    loading.value = false;
  } finally {
    loadingMessage();
  }
}

onMounted(() => {
  getListOfCampaigns();
  // setInterval(checkTimeAndRun, 60000);
});

watch(campaigns, (newCampaigns, oldCampaigns) => {
    if (newCampaigns.length > 0 && newCampaigns !== oldCampaigns) {
      getInformationAboutCampaigns(newCampaigns);
      updateCampaignBudgets(newCampaigns);
      updateCampaignCtr(newCampaigns);
    }
  }
);

const launchedCampaigns = ref([]);

// Функция для обновления состояния enabled
function updateCampaignEnabled(id, enabled) {
  const index = campaigns.value.findIndex(c => c.advertId === id);
  if (index !== -1) {
    // Перезаписываем объект кампании целиком
    campaigns.value[index] = {
      ...campaigns.value[index],
      enabled: enabled,  // Обновляем свойство enabled
    };
  }

  const campaign = launchedCampaigns.value.find(c => c.advertId === id);

  if (!campaign && enabled) {
    launchedCampaigns.value.push({
      advertId: id,
    })
  } else if (campaign && !enabled) {
    const index = launchedCampaigns.value.findIndex(c => c.advertId === id);
    if (index !== -1) {
      launchedCampaigns.value.splice(index, 1); // Удаляем элемент по индексу
    }
  }
}

// const groupedCampaigns = ref({});

// watch(launchedCampaigns, (newCampaigns) => {
//   const newCampaignsIds = newCampaigns.map(c => c.advertId);
//   const matchingCampaigns = campaigns.value.filter(c => newCampaignsIds.includes(c.advertId));
//   groupedCampaigns.value = matchingCampaigns.reduce((groups, campaign) => {
//     const interval = campaign.interval;
//     if (!groups[interval]) groups[interval] = [];
//     groups[interval].push(campaign);
//     return groups;
//   }, {});
// });
//
// // Функция для запуска задач
// function start(campaignsGroup) {
//   getInformationAboutCampaigns(campaignsGroup);
//   updateCampaignBudgets(campaignsGroup);
//   updateCampaignCtr(campaignsGroup);
// }
//
// // Функция проверки времени и запуска задач
// function checkTimeAndRun() {
//   const now = new Date();
//   const minutes = now.getMinutes();
//   for (const interval in groupedCampaigns.value) {
//     const campaignsGroup = groupedCampaigns.value[interval];
//     if (minutes % interval === 0) {
//       start(campaignsGroup);
//     }
//   }
// }

watch(launchedCampaigns, (newCampaigns) => {
  const newCampaignsIds = newCampaigns.map(c => c.advertId);
  const matchingCampaigns = campaigns.value.filter(c => newCampaignsIds.includes(c.advertId));

  function groupByInterval(campaigns) {
    const groups = {
      15: [],
      20: [],
      30: []
    };

    campaigns.forEach((campaign) => {
      if (groups[campaign.interval]) {
        groups[campaign.interval].push(campaign);
      }
    });

    return groups;
  }

  const result = groupByInterval(matchingCampaigns);
  console.log(result);

  // const groupedCampaigns = matchingCampaigns.reduce((groups, campaign) => {
  //   const interval = campaign.interval;
  //   if (!groups[interval]) {
  //     groups[interval] = [];
  //   }
  //   groups[interval].push(campaign);
  //   return groups;
  // }, {});
  //
  // console.log("groupedCampaigns", groupedCampaigns);

// Функция start для запуска задач
//   function start(campaignsGroup) {
//     getInformationAboutCampaigns(campaignsGroup);
//     updateCampaignBudgets(campaignsGroup);
//     updateCampaignCtr(campaignsGroup);
//     // console.log(new Date().getMinutes(), campaignsGroup);
//     // campaignsGroup.forEach(campaign => {
//     //   console.log(`Запуск для объекта с id: ${campaign.id}, интервал: ${campaign.interval}`);
//     //   // Здесь разместите логику запуска для каждого объекта
//     // });
//   }

// Функция, которая проверяет текущее время
//   function checkTimeAndRun() {
//     const now = new Date();
//     const minutes = now.getMinutes();
//
//     // Проходим по всем группам интервалов
//     for (const interval in groupedCampaigns) {
//       const campaignsGroup = groupedCampaigns[interval];
//
//       // Если текущее время кратно интервалу, запускаем start для этой группы
//       if (minutes % interval === 0) {
//         start(campaignsGroup);
//       }
//     }
//   }

// Устанавливаем интервал проверки каждую минуту
//   setInterval(checkTimeAndRun, 60000);
}, { deep: true });


// Функция для обновления количества показов
const updateCampaignImpressions = (id, value) => {
  const index = campaigns.value.findIndex(c => c.advertId === id);
  if (index !== -1) {
    // Перезаписываем объект кампании целиком
    campaigns.value[index] = {
      ...campaigns.value[index],
      impressions: value,  // Обновляем свойство impressions
    };
  }
};

// Функция для обновления интервала
const updateCampaignInterval = (id, value) => {
  const index = campaigns.value.findIndex(c => c.advertId === id);
  if (index !== -1) {
    // Перезаписываем объект кампании целиком
    campaigns.value[index] = {
      ...campaigns.value[index],
      interval: value,
      // interval: { // Обновляем свойство interval
      //   value: value,
      //   repeat: value && 60 / value,
      // },
    };
  }
};

const updateCampaignUpdated = (id, value) => {
  const index = campaigns.value.findIndex(c => c.advertId === id);
  if (index !== -1) {
    // Перезаписываем объект кампании целиком
    campaigns.value[index] = {
      ...campaigns.value[index],
      updated: value,
    };

    updateCampaignBudgets([id]);
  }
}
</script>

<template>
  <a-config-provider :locale="ruRu">
    <a-form ref="form" layout="vertical">
      <a-row :gutter="24">
        <a-col :span="12">
          <a-form-item label="Компания">
            <a-select
              v-model:value="companySelected"
              :options="companyOptions"
            />
          </a-form-item>
        </a-col>
        <a-col :span="6">
          <a-button
            @click="getListOfCampaigns"
            :disabled="disabledUpdateCampaign"
            style="margin-top: 30px;"
            block
          >
            <ReloadOutlined /> Обновить кампании
          </a-button>
        </a-col>
      </a-row>
    </a-form>

    <div class="campaign-list">
      <CardCampaign
        v-for="campaign in campaigns"
        :key="campaign.advertId"
        :campaign="campaign"
        :loading="loading"
        @updateEnabled="updateCampaignEnabled(campaign.advertId, $event)"
        @updateImpressions="updateCampaignImpressions(campaign.advertId, $event)"
        @updateInterval="updateCampaignInterval(campaign.advertId, $event)"
        @updateUpdated="updateCampaignUpdated(campaign.advertId, $event)"
      />
    </div>
  </a-config-provider>
</template>

<style scoped>
.campaign-list {
  display: flex;
  flex-direction: column;
  row-gap: 20px;
}
</style>
