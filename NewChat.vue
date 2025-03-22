<template>
  <div class="flex-1 overflow-hidden">
    <div class="relative h-full">
      <div class="scrollbox react-scroll-to-bottom">
        <div class="flex flex-col text-sm">
          <div
            class="w-full mx-auto md:h-full md:flex md:flex-col px-6"
          >
            <div class="px-6 my-4">
              <img src="@/assets/midm.png" height="15"/>
              <div class="flex justify-center mb-4">
                <img src="@/assets/product_name.png"/>
              </div>
            </div>
            <div class="md:flex items-start gap-3.5 mb-8 md:mb-auto"
              v-if="aiModelStore.isChat"
            >
              <gen-card class="flex flex-col flex-1 mx-6"
                title="GPT - 3.5"
                @click="selectAi('azure')"
                :selected="chatStore.aiModel == 'azure'"
                smallTitle="MS Azure"
                :modelOpen='true'
              >
                <template v-slot:icon>
                  <div class="mt-auto">
                    <img src="@/assets/azure_img.png" width="18" height="18" />
                  </div>
                </template>
                GPT-3.5 excels in natural language processing, enabling versatile 
                applications such as efficient information retrieval, 
                user support, and language translation.
              </gen-card>
              <gen-card class="flex flex-col flex-1 mx-6"
                title="GPT - 4"
                :selected="chatStore.aiModel == 'gpt4'"
                smallTitle="MS Azure"
                :modelOpen='false'
              >
                <template v-slot:icon>
                  <div class="mt-auto">
                    <img src="@/assets/gpt4_img.png" width="18" height="18" />
                  </div>
                </template>
                <!-- GPT-4 offers superior language processing, elevating productivity 
                and automation in diverse fields for future business 
                and technological progress. -->
                Comming Soon.
              </gen-card>
              <gen-card class="flex flex-col flex-1 mx-6" 
                title="DALL·E 3" 
                :modelOpen='false'
                :selected="chatStore.aiModel == 'dalle3'"
                smallTitle="MS Azure"
              >
                <template v-slot:icon>
                  <div class="mt-auto">
                    <img src="@/assets/dalle3_img.png" width="18" height="18" />
                  </div>
                </template>
                Comming Soon.
                <!-- DALL·E 3 understands significantly more nuance and detail 
                than our previous systems, allowing you to easily translate 
                your ideas into exceptionally accurate images. -->
              </gen-card>
            </div>
            <div class="md:flex items-start gap-3.5 mb-8 md:mb-auto"
              v-if="aiModelStore.isChat"
            >
              <gen-card class="flex flex-col flex-1 mx-6" 
                title="Mi:dm"
                smallTitle="믿:음 Playground"
                @click="kai = true"
                :modelOpen='true'
              >
                <template v-slot:icon>
                  <div style="margin-top: 0.330rem;">
                    <img src="@/assets/KTlogo.png" height="15" />
                  </div>
                </template>
                  KAI Studio에서 KT Mi:dm 모델을 사용해 볼 수 있습니다.
                <!-- <br>
                <br>
                <br>
                <br>
                <br>
                <br>
                <br>
                <p class="bottomText">
                  
                </p> -->
                <!-- <template v-slot:bottomText>
                  <div>
                    채팅 대화는 KT Mi:dm LLM 학습에 활용될 수 있습니다.
                  </div>
                </template> -->
              </gen-card>
              <gen-card class="flex flex-col flex-1 mx-6"
                title="Bard"
                :selected="chatStore.aiModel == 'bard'"
                smallTitle="Google VertexAI"
                :modelOpen='false'
              >
                <template v-slot:icon>
                  <div class="mt-auto">
                    <img src="@/assets/bard_ani.gif" width="18" height="18" />
                  </div>
                </template>
                Comming Soon.
              </gen-card>
              <gen-card class="flex flex-col flex-1 mx-6" 
                title="Imagen" 
                :modelOpen='false' 
                smallTitle="(English only) Google VisionAI"
                :selected="chatStore.aiModel == 'image'"
              >
                <template v-slot:icon>
                  <div class="mt-auto">
                    <img src="@/assets/vision_img.png" width="18" height="18" />
                  </div>
                </template>
                Comming Soon.
              </gen-card>
            </div>
            <div class="h-28 md:h-32 flex-shrink-0"></div>
          </div>
        </div>
      </div>
    </div>
    <custom-confirm
      title="안내" 
      content="KAI Studio로 이동합니다."
      :noneClick="() => { kai = false }"
      :successClick="gotoKai"
      v-if="kai"
    />
  </div>
</template>

<script setup lang="ts">
import GenCard from "@/components/GenCard.vue";
import CustomConfirm from "@/components/CustomConfirm.vue";
import { onMounted, ref } from "vue";
import { useGenAiChatDataStore, useGenAiModelsStatusStore } from "@/store";
import { v4 as uuidv4 } from 'uuid';

const aiModelStore = useGenAiModelsStatusStore();
const chatStore = useGenAiChatDataStore();
const kai = ref(false);

// 모델 선택
const selectAi = (aiModel : string) => {
  chatStore.aiModel = aiModel;
}

const gotoKai = () =>{
  kai.value = false;
  window.open('https://gotoKai.com/');
}

onMounted(() => {
  const randomUUID : string = uuidv4();
  chatStore.chatData = [];
  chatStore.pdfSrc = '';
  chatStore.pdfMetaData = [];
  chatStore.selectedPdf = '';
  chatStore.pdfPage = 1;
  localStorage.removeItem('pdfjs.history');
  sessionStorage.setItem('sessionId', randomUUID);
  chatStore.sessionId = randomUUID;
})

</script>
<style scoped lang="scss">
.bottomText{
  opacity: 0.6;
  text-align: left;
}
</style>