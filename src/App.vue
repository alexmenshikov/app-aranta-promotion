<script setup>
import axios from "axios";
import CardCampaign from "./components/CardCampaign.vue";
import {computed, ref, watch} from "vue";
import {
  Col as ACol,
  ConfigProvider as AConfigProvider,
  Form as AForm,
  FormItem as AFormItem,
  Row as ARow,
  Select as ASelect,
} from "ant-design-vue";
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
];

const transformedCompanyOptions = computed(() => {
  return companyArray.map(company => ({
    label: company.name,
    value: JSON.stringify(company)
  }));
});

const companyOptions = ref([]);
const companySelected = ref(JSON.stringify(companyArray[0]));

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

const listCampaignsIds = ref({ ids: [], objectIds: [] });
const listInformationAboutCampaigns = ref([]);

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
  return axios
    .get(`https://advert-api.wildberries.ru/adv/v1/budget?id=${id}`, {
      headers: {
        Authorization: `${transformedCompanySelected.value.apiToken}`
      }
    })
    .then(response => response.data.total)
    .catch(error => {
      console.log("getCampaignBudget: ", error);
      return null;
    });
}

async function updateCampaignBudgets(campaignIds) {
  // Перебираем каждый элемент newIds и обновляем budget
  for (const id of campaignIds) {
    const budget = await getCampaignBudget(id);

    // Находим объект в listInformationAboutCampaigns с совпадающим advertId
    const campaign = listInformationAboutCampaigns.value.find(
      item => item.advertId === id
    );

    if (campaign) {
      campaign.budget = budget;
    }
  }
}

// Функция для получения CTR
async function updateCampaignCtr(campaignIds) {
  axios
    .post("https://advert-api.wildberries.ru/adv/v2/fullstats", campaignIds,{
      headers: {
        Authorization: `${transformedCompanySelected.value.apiToken}`
      }
    })
    .then(response => {
      for (const element of response.data) {
        // Находим объект в listInformationAboutCampaigns с совпадающим advertId
        const campaign = listInformationAboutCampaigns.value.find(
          item => item.advertId === element.advertId
        );

        if (campaign) {
          campaign.ctr = element.ctr;
          campaign.views = element.views;
        }
      }
    })
    .catch(error => {
      console.log("getCampaignCtr: ", error);
    });
}

// Функция для получения информации о компаниях
async function getInformationAboutCampaigns(campaignIds) {
  try {
    const response = await axios.post("https://advert-api.wildberries.ru/adv/v1/promotion/adverts", campaignIds, {
        headers: {
          Authorization: `${transformedCompanySelected.value.apiToken}`,
        },
      }
    );

    listInformationAboutCampaigns.value = response.data.map(item => ({
      name: item.name,
      advertId: item.advertId,
      type: {
        original: item.type,
        convert: convertCampaignsType(item.type),
      },
      status: {
        original: item.status,
        convert: convertCampaignsStatus(item.status),
      },
      cpm: item.type === 8 ? item.autoParams.nmCPM[0].cpm : item.unitedParams[0].searchCPM,
      budget: null,
      ctr: null,
      views: null,
    }));
  } catch (error) {
    console.error("getInformationAboutCampaigns", error);
  }
}

// Следим за изменениями в listCampaignsIds, если нужно автоматически вызывать вторую функцию
watch(() => listCampaignsIds.value.ids, (newIds) => {
    if (newIds.length > 0) {
      // Вызов getInformationAboutCampaigns с заполненными данными
      getInformationAboutCampaigns(newIds);
      updateCampaignBudgets(newIds);
    }
  }
);

watch(() => listCampaignsIds.value.objectIds, (newIds) => {
    if (newIds.length > 0) {
      // Вызов getInformationAboutCampaigns с заполненными данными
      // getInformationAboutCampaigns(newIds);
      // updateCampaignBudgets(newIds);
      updateCampaignCtr(newIds);
    }
  }
);

  // axios
  //   .post("https://advert-api.wildberries.ru/adv/v1/promotion/adverts", campaignIds,{
  //     headers: {
  //       Authorization: `${transformedCompanySelected.value.apiToken}`
  //     }
  //   })
  //   .then(response => {
  //     listInformationAboutCampaigns.value = response.data.map(item => ({
  //       name: item.name,
  //       advertId: item.advertId,
  //       type: {
  //         original: item.type,
  //         convert: convertCampaignsType(item.type),
  //       },
  //       status: {
  //         original: item.status,
  //         convert: convertCampaignsStatus(item.status),
  //       },
  //       cpm: item.type === 8 ? item.autoParams.nmCPM[0].cpm : item.unitedParams[0].searchCPM,
  //       budget: null,
  //       ctr: null,
  //     }));
  //   })
  //   .catch(error => {
  //     console.log(error);
  //     // message.error('Ошибка при загрузке отзывов!');
  //     // loading.value = false;
  //   });
// }

// Функция для обновления CTR
// async function updateCampaignCtr() {
//   const campaignIds = listInformationAboutCampaigns.value.map(campaign => ({
//     id: campaign.advertId
//   }));
//
//   const response = await getCampaignCtr(campaignIds);
//
//   console.log("response", response);
  // if (response) {
    // listInformationAboutCampaigns.value.forEach(campaign => {
    //   const matchingCampaign = response.find(item => item.advertId === campaign.advertId);
    //
    //   console.log("campaign", campaign);
    //   if (matchingCampaign) {
    //     campaign.ctr = matchingCampaign.ctr;
    //   }
    // });

    // Создаем новый массив с обновленными данными
    // const updatedCampaigns = listInformationAboutCampaigns.value.map(campaign => {
    //   const matchingCampaign = response.find(item => item.advertId === campaign.advertId);
    //   if (matchingCampaign) {
    //     return { ...campaign, ctr: matchingCampaign.ctr, test: "test" };
    //   }
    //   return campaign;
    // });

    // Обновляем массив целиком
    // listInformationAboutCampaigns.value = updatedCampaigns
  // }
// }

// watch(listInformationAboutCampaigns, async (newCampaigns) => {
//   await Promise.all(newCampaigns.map(async (item, index) => {
//     listInformationAboutCampaigns.value[index].budget = await getCampaignBudget(item.advertId);
//   }));
//
//   // const campaignIds = listInformationAboutCampaigns.value.map((item) => ({
//   //   id: item.advertId,
//   // }))
//
//   await updateCampaignCtr();
// });

// watch(listInformationAboutCampaigns, (newCampaigns) => {
//   newCampaigns.forEach((item, index) => {
//     getCampaignBudget(item.advertId).then(budget => {
//       listInformationAboutCampaigns.value[index].budget = budget;
//     });
//   });
// });

// Функция для получения кампаний
async function getListOfCampaigns() {
  listCampaignsIds.value = { ids: [], objectIds: [] };

  try {
    const response = await axios.get("https://advert-api.wildberries.ru/adv/v1/promotion/count", {
      headers: {
        Authorization: `${transformedCompanySelected.value.apiToken}`,
      },
      }
    );

    const listIds = response.data.adverts.flatMap((advert) =>
      advert.advert_list.map((item) => item.advertId)
    );

    const listObjectIds = response.data.adverts.flatMap((advert) =>
      advert.advert_list.map((item) => ({ id: item.advertId }))
    );

    listCampaignsIds.value.ids = listIds;
    listCampaignsIds.value.objectIds = listObjectIds;
  } catch (error) {
    console.error("getListOfCampaigns", error);
  }
  // axios
  //   .get("https://advert-api.wildberries.ru/adv/v1/promotion/count", {
  //     headers: {
  //       Authorization: `${transformedCompanySelected.value.apiToken}`
  //     }
  //   })
  //   .then(response => {
  //     listIds = response.data.adverts.flatMap(advert =>
  //       advert.advert_list.map(item => item.advertId)
  //     );
  //
  //     listObjectIds = response.data.adverts.flatMap(advert =>
  //       advert.advert_list.map(item => ({ id: item.advertId }))
  //     );
  //
  //     console.log("listIds ", listIds);
  //     console.log("listObjectIds ", listObjectIds);
  //
  //     listCampaignsIds.value.ids = listIds;
  //     listCampaignsIds.value.objectIds = listObjectIds;
  //     // getInformationAboutCampaigns(listCampaignsIds.value);
  //     // loading.value = false;
  //   })
  //   .catch(error => {
  //     console.log("getListOfCampaigns", error);
  //     // message.error('Ошибка при загрузке отзывов!');
  //     // loading.value = false;
  //   });
}

getListOfCampaigns();
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
      </a-row>
    </a-form>
  </a-config-provider>

<!--  <HelloWorld msg="Vite + Vue" />-->
  <div class="campaign-list">
<!--    <div>id - {{ campaign.advertId }}</div>-->
<!--    <div>название - {{ campaign.name }}</div>-->
<!--    <div>тип компании - {{ campaign.type.convert }}</div>-->
<!--    <div>статус - {{ campaign.status.convert }}</div>-->
<!--    <div>текущая ставка {{ campaign.currentRate }}</div>-->
    <CardCampaign v-for="campaign in listInformationAboutCampaigns" :key="campaign.advertId" :campaign="campaign" />
  </div>
</template>

<style scoped>
.campaign-list {
  display: flex;
  flex-direction: column;
  row-gap: 20px;
}
</style>
