<template>
  <div class="flex flex-col gap-2 pb-2 text-sm">
    <div class="relative">
      <history-list-group v-for="(group, groupIdx) in chatStore.chatTitleData"
      :key="groupIdx" 
      :title="group.chatGroup == 0 ? '오늘' : 
      group.chatGroup == -1 ? '어제' : 
      group.chatGroup <= -2 && group.chatGroup >= -6 ? '7일이내' : 
      group.chatGroup <= -7 && group.chatGroup >= -30 ? '30일이내' : '오래 전'"
      >
        <history-list-item
          v-for="(data, listIdx) in group.titleList"
          :key="listIdx"
          @click="clickTitle(data.chatTime, data.sessionId, data.aiModel, data.embeddingId)"
          :aiModel="data.aiModel"
          :groupIndex="groupIdx"
          :listIndex="listIdx"
          @editConfirm="editHistoryTitle"
          :selected="data.sessionId == chatStore.sessionId"
          :deletefx="(()=>dialogFlag = true)"
        >
          {{ data.title }}
        </history-list-item>
      </history-list-group>
    </div>
    <custom-confirm
      title="채팅 삭제" 
      content="정말 삭제하시겠습니까?"
      :noneClick="(()=>dialogFlag = false)"
      :successClick="deleteButtonClick"
      v-if="dialogFlag"
    />
  </div>
  <!-- </div> -->
</template>

<script setup lang="ts">
import { useRouter } from "vue-router";
import CustomConfirm from "@/components/CustomConfirm.vue";
import {
  useWindowStatusStore,
  useGenAiSideStatusStore,
  useGenAiChatDataStore,
  useGenAiModelsStatusStore,
} from "@/store";
import { ChatTitle } from "@/type/store";
import { ref, onMounted } from "vue";

const router = useRouter();
const winStore = useWindowStatusStore();
const aiSideStore = useGenAiSideStatusStore();
const chatStore = useGenAiChatDataStore();
const aiModelStore = useGenAiModelsStatusStore();

const dialogFlag = ref(false);

// 지원중단 모델
const unusableModel = ref<string[]>([]);
const unusableMessage = ref('');
const currentAimodel = ref('');

onMounted(() => {
  // 채팅 히스토리 가져와
  getChatTitle();
  // 지원중단 모델 확인
  getUnusableModel();
  dialogFlag.value = false;
});

// 히스토리 그룹화를 위한 오늘 날짜
const now = new Date();
const nowDate = new Date(
  now.getFullYear(),
  now.getMonth(),
  now.getDate()
);

// 히스토리 타이틀 수정
const editHistoryTitle = (data : {groupIdx : number, 
  listIdx : number, newTitle : string}) => {
  const sendData = {
    sessionId : chatStore.sessionId,
    chatName : data.newTitle
  };
  
  instance.post('/chat/update', sendData)
    .then(res => {
      chatStore.titleEditFlag = false;
      chatStore.chatTitleData[data.groupIdx].titleList[data.listIdx].title = data.newTitle;
    })
    .catch(err => {
      console.log(err);
      chatStore.alertContent = `에러가 발생하였습니다.\n관리자에게 문의해주세요.`;
      chatStore.openAlert = true;
    })
}

// 히스토리 삭제 버튼 클릭 함수
const deleteButtonClick = () => {
  dialogFlag.value = true;

  let sendData : object = {};

  if (chatStore.aiModel == 'embed') {
    sendData = {
      sessionId : chatStore.sessionId,
      embeddingId : chatStore.embeddingId,
      type : 'embed'
    }
  } else {
    sendData = {
      sessionId : chatStore.sessionId,
      type : chatStore.aiModel
    }
  }
    
  instance.post('chat/delete', sendData)
    .then((res)=>{
      getChatTitle();
      aiModelStore.isEmbed = false; // 임베드 컴포넌트 없애주고
      chatStore.embeddingId = null; // 임베딩아이디 초기화해주고
      chatStore.chatData = [];
      chatStore.aiModel = 'azure';
      sessionStorage.removeItem('embeddingId');
      aiModelStore.isChat = true;
      aiSideStore.inputFlag = true;
      router.push('/ai/new');
      
      dialogFlag.value = false;
    })
    .catch(err => {
      console.log(err);
      chatStore.alertContent = `에러가 발생하였습니다.\n관리자에게 문의해주세요.`;
      chatStore.openAlert = true;
      dialogFlag.value = false;
    })
}

const clickTitle = (idx: number, sessionId: string, aiModel: string, embeddingId: string | null) => {
  
  if (unusableModel.value.includes(aiModel)) {
    aiSideStore.inputFlag = false;
    chatStore.alertTitle = '서비스 종료';
    chatStore.alertContent = unusableMessage.value;
    chatStore.openAlert = true;
  } else {
    aiSideStore.inputFlag = true;
  }

  chatStore.titleEditFlag = false;
  aiModelStore.selectedAiModelName = '';
  chatStore.chatData = [];
  chatStore.pdfImageData = [];
  chatStore.pdfMetaData = [];
  chatStore.pdfSrc = '';
  chatStore.selectedPdf = '';
  chatStore.pdfPage = 1;
  if (winStore.isMobileMode) {
    aiSideStore.hide();
  }
  chatStore.embeddingId = embeddingId;
  chatStore.aiModel = aiModel;

  if (embeddingId) {
    aiModelStore.isEmbed = true;
  }else {
    aiModelStore.isEmbed = false;
  }
  getChatContents(idx, sessionId, aiModel, embeddingId);
};

const getChatTitle = () => {
  
  chatStore.chatTitleData = [];
  instance.get(`/chat/chatHistory`)
    .then((res) => {

      const chatTitleMap = new Map<number, ChatTitle[]>();
      
      res.data.forEach(
        (item: {
          id: string;
          chatName: string;
          chatType: string;
          embeddingId: string;
          chatTime: Date;
        }) => {
          const chatTimeDate = new Date(
            new Date(item.chatTime).getFullYear(),
            new Date(item.chatTime).getMonth(),
            new Date(item.chatTime).getDate()
          );
          // chatTime 기준 현재와 날짜 차이 계산
          let chatDateDiff = (chatTimeDate.getTime() - nowDate.getTime()) / (1000*60*60*24);
          
          // 그룹화를 위한 값 재할당
          if(chatDateDiff <= -2 && chatDateDiff >= -6) {
            chatDateDiff = -6;
          } else if (chatDateDiff <= -7 && chatDateDiff >= -30) {
            chatDateDiff = -30;
          } else if (chatDateDiff <= -31) {
            chatDateDiff = -60;
          }
          const groupData = chatTitleMap.get(chatDateDiff);
          // 그룹에 따른 데이터 push
          if (groupData) {
            groupData.push({
              title: item.chatName,
              sessionId: item.id,
              aiModel: item.chatType,
              embeddingId: item.embeddingId,
              chatTime: new Date(item.chatTime).getTime()
            })
          } else {
            chatTitleMap.set(chatDateDiff, [
              {
                title: item.chatName,
                sessionId: item.id,
                aiModel: item.chatType,
                embeddingId: item.embeddingId,
                chatTime: new Date(item.chatTime).getTime()
              }
            ])
          }
        }
        
      );
      // Map 객체에 있는 값 store 배열에 넣어주기
      chatTitleMap.forEach((groupData, chatDateDiff) => {
        // 그룹에 따른 내림차순 정렬
        groupData.sort((a, b) => {
          return b.chatTime - a.chatTime;
        });

        chatStore.chatTitleData.unshift({
          chatGroup : chatDateDiff,
          titleList : groupData
        });
      });

      // 마지막으로 내림차순 정렬
      chatStore.chatTitleData.sort((a, b) => {
        return b.chatGroup - a.chatGroup;
      });
      
    })
    .catch((err) => {
      console.log(err);
    });
};
// 채팅 내용 가져오는 api
const getChatContents = (idx: number, sessionId: string, aiModel: string, embeddingId: string | null) => {
  chatStore.sessionId = sessionId;
  chatStore.aiModel = aiModel;

  sessionStorage.setItem("sessionId", sessionId);

    instance.get(`/chat/chatHistoryDetail/${sessionId}`)
      .then((res) => {
        chatStore.chatData = [];
        if (aiModel !== 'image') {
          res.data.forEach(
            (item: {
              question: string;
              result: { answer: string; role: string; embedding_id: string };
            }) => {
              chatStore.chatData.push(
                {
                  role: "user",
                  content: item.question,
                  sessionId: sessionId,
                  embeddingId: item.result.embedding_id,
                },
                {
                  role: "assistant",
                  content: item.result.answer,
                  sessionId: sessionId,
                  embeddingId: item.result.embedding_id,
                }
              );
              sessionStorage.setItem("embeddingId", item.result.embedding_id);
            }
          );
          // 모델이 embed일 경우
          if (aiModel == 'embed' && embeddingId) {
            // 참조페이지 가져오기
            getPdfPage().then((ress) => {
              ress.pdf_metadata.map((item : {pdf : string, pageIndex : number}) => {
                chatStore.pdfMetaData.push({
                  pdfSrc : item.pdf,
                  pdfPage : item.pageIndex
                })
              })
              let responseSrc = ress.pdf_metadata[0].pdf;
              chatStore.pdfPage = ress.pdf_metadata[0].pageIndex + 1;
              chatStore.selectedPdf = formatter.splitFileName(responseSrc);
              chatStore.selectedRefer = 0;
            });

            // pdf 정보 가져오기
            getEmbedInfo(embeddingId).then(ress => {
              chatStore.setPdfState(1);
              chatStore.setImageData(ress.images);

              let responseSrc = ress.pdf_metadata[0].pdf;
              chatStore.pdfSrc = '/img/gen/' + responseSrc.substring(responseSrc.search('genai'));

              winStore.loading = false;
            })
          }
        // 모델이 image일 경우
        } else {  
          res.data.forEach(
            (item: {
              question: string,
              result: { role: string; image_list: Array<string> }
            }) => {
              chatStore.chatData.push(
                {
                  role: "user",
                  content: item.question,
                  sessionId: sessionId,
                  embeddingId: null,
                },
                {
                  role: "assistant",
                  content: item.result.image_list[0] == 'IMAGE_GENERATION_FAILED'
                    ? 'Image creation failed. Please try again with a different keyword. (English only)'
                    : '',
                  imageList: item.result.image_list[0] == 'IMAGE_GENERATION_FAILED'
                    ? [] 
                    : item.result.image_list,
                  sessionId: sessionId,
                  embeddingId: null,
                }
              );
            }
          );
        }

      })
      .catch((err) => {
        console.log(err);
      });
  

  router.push("/ai/c/code" + idx);
};

const getUnusableModel = async () => {
  instance.post('/check')
    .then(res => {
      currentAimodel.value = chatStore.aiModel ?? 'azure';
      res.data.unusable.forEach((item: string) => {
        unusableModel.value.push(item);
      });
      unusableMessage.value = res.data.error_message;

      if (unusableModel.value.includes(currentAimodel.value)) {
        aiSideStore.inputFlag = false;
        chatStore.alertTitle = '서비스 종료';
        chatStore.alertContent = unusableMessage.value;
        chatStore.openAlert = true;
      }
    })
    .catch(err => {
      console.log(err);
    })
}

</script>
<style lang="scss" scoped>
.titleInput {
  color: #b3bcd0;
  border: 1px solid #ffffff;
  outline: none;
  height: 1.25rem;
}

</style>