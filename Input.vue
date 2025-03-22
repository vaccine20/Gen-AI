<template>
  <div
    class="stretch flex flex-row gap-3 mx-2 md:mx-4 lg:mx-auto lg:max-w-2xl xl:max-w-3xl"
  >
    <div
      class="relative flex h-full flex-1 items-stretch md:flex-col"
      role="presentation"
    >
      <div class="btn-area" v-if="winStore.isDesktopMode && !(isChatting == 'new') && 
      (chatStore.aiModel == 'azure' || chatStore.aiModel == 'gpt4')">
        <div
          class="h-full flex ml-1 md:w-full md:m-auto md:mb-2 gap-0 md:gap-2 justify-center"
        >
          <button
            class="btn relative btn-neutral -z-0 border-0 md:border"
            v-if="!streaming"
            @click="handleRegen"
          >
            <div class="flex w-full gap-2 items-center justify-center">
              <reload-icon />
              Regenerate
            </div>
          </button>
          <button
            class="btn relative btn-neutral -z-0 border-0 md:border"
            v-else
            @click="handleStop"
          >
            <div class="flex w-full gap-2 items-center justify-center">
              <close-icon/>
              Stop generating
            </div>
          </button>
        </div>
      </div>
      <div
        class="flex flex-col w-full py-[10px] flex-grow md:py-4 md:pl-4 relative border-3 bg-white rounded-[20px]"
      >
        <textarea
          type="text"
          ref="textareaRef"
          id="prompt-textarea"
          tabindex="0"
          data-id="root"
          rows="1"
          autofocus
          placeholder="궁금하신 것을 입력해보세요."
          v-model="askText"
          @input="handleAskText"
          @keydown.enter="askSendEnter"
          class="m-0 w-full resize-none border-0 bg-transparent p-0 pr-10 md:pr-12 pl-3 md:pl-0 text-scrollbox"
          style="max-height: 200px; height: 24px; overflow-y: hidden"
        ></textarea>
        <button
          v-if="!streaming"
          @click="askSend"
          :disabled="askText.trim() == ''"
          class="absolute p-1 rounded-md md:bottom-3 md:p-2 md:right-3 right-2 disabled:text-gray-400 bottom-1.5 transition-colors disabled:opacity-40"
        >
          <span data-state="closed">
            <send-icon />
          </span>
        </button>
        <div
          v-else
          class="absolute p-1 rounded-md md:p-2 md:right-3 right-2 disabled:text-gray-400 bottom-1.5 transition-colors"
        >
          <v-progress-circular
            indeterminate
            color="#64dfdd"
          ></v-progress-circular>
        </div>
      </div>
      <!-- <div class="" v-if="winStore.isMobileMode">
        <div
          class="h-full flex ml-1 md:w-full md:m-auto md:mb-2 gap-0 md:gap-2 justify-center"
        >
          <button class="btn relative -z-0 border-0 md:border" as="button">
            <div class="flex w-full gap-2 items-center justify-center">
              <reload-icon />
            </div>
          </button>
        </div>
      </div> -->
    </div>
  </div>
</template>

<script setup lang="ts">
import {
  useWindowStatusStore,
  useGenAiChatDataStore,
} from "@/store";
import router from "@/router";
import { ref, watchEffect } from "vue";

const winStore = useWindowStatusStore();
const chatStore = useGenAiChatDataStore();

const streaming = ref(false);

const regenYn = ref('N');

const askText = ref('');
const textareaRef = ref<HTMLTextAreaElement | null>(null);
const isChatting = ref(router.currentRoute.value.name);

watchEffect(() => {
  isChatting.value = router.currentRoute.value.name;
})

const handleAskText = (e: Event) => {
  askText.value = (e.target as HTMLInputElement).value;

  const textarea = textareaRef.value;
  if (!textarea) {
    return;
  }

  textarea.style.height = "auto";
  textarea.style.height = textarea.scrollHeight + "px";
  textarea.style.overflowY = "auto";
};

const askSendEnter = (e: KeyboardEvent) => {

  if (!e.shiftKey && !streaming.value) {
    askSend(e);
  }
  const textarea = textareaRef.value;
  if (!textarea) {
    return;
  }

  textarea.style.height = "24px";
};

const askSend = async (e: Event) => {
  
  e.preventDefault();
  if (askText.value == null || askText.value == undefined) {
    return;
  } else if (askText.value.trim() !== "") {
    streaming.value = true;
    const sessionId = sessionStorage.getItem("sessionId");
    chatStore.sessionId = sessionId ?? '';

    if (router.currentRoute.value.path == "/gen-ai/new"
      && chatStore.aiModel !== 'azure' && chatStore.aiModel !== 'gpt4' && chatStore.aiModel !== 'embed') {
        
        if (chatStore.chatTitleData.length == 0) {
        chatStore.chatTitleData.unshift({
          chatGroup: 0,
          titleList: [
            {
              title: askText.value,
              sessionId: chatStore.sessionId,
              aiModel: chatStore.aiModel,
              embeddingId: chatStore.embeddingId,
              chatTime: new Date().getTime(),
            },
          ],
        });
      } else if (
        chatStore.chatTitleData.length !== 0 &&
        chatStore.chatTitleData[0].chatGroup !== 0
      ) {
        chatStore.chatTitleData.unshift({
          chatGroup: 0,
          titleList: [
            {
              title: askText.value,
              sessionId: chatStore.sessionId,
              aiModel: chatStore.aiModel,
              embeddingId: chatStore.embeddingId,
              chatTime: new Date().getTime(),
            },
          ],
        });
      } else {
        chatStore.chatTitleData[0].titleList.unshift({
          title: askText.value,
          sessionId: chatStore.sessionId,
          aiModel: chatStore.aiModel,
          embeddingId: chatStore.embeddingId,
          chatTime: new Date().getTime(),
        });
      }
    }

    chatStore.chatData.push({
      role: "user",
      content: askText.value,
      sessionId: chatStore.sessionId,
      embeddingId: chatStore.embeddingId,
    });
    
    
    if (chatStore.aiModel !== 'image') {
      chatStore.chatData.push({
        role: "assistant",
        content: "",
        sessionId: chatStore.sessionId,
        embeddingId: chatStore.embeddingId
      });
    } else {
      chatStore.chatData.push({
        role: "assistant",
        content: "",
        sessionId: chatStore.sessionId,
        embeddingId: chatStore.embeddingId,
        imageList: []
      });
    }
    
    const messageData = [];
    messageData.push(chatStore.chatData[0]);
    for (let i = 1; chatStore.chatData.length > 1 && i < chatStore.chatData.length; i++) {
      if (chatStore.chatData[i].isError) {
        messageData.pop();
        continue;
      }
      messageData.push(chatStore.chatData[i]);
    }

    const sendData: {
      messages: Array<object>,
      type: string,
      sessionId: string,
      embeddingId: string | null,
      regen: string,
      chatOrder: number,
    } = {
      // messages는 문맥대화를 위해 최근 대화내용 세 쌍 보내기
      messages:
        chatStore.chatData.length <= 6
          ? messageData
          : messageData.slice(messageData.length - 6),
      type: chatStore.aiModel,
      sessionId: chatStore.sessionId,
      embeddingId: chatStore.embeddingId,
      regen: regenYn.value,
      chatOrder: chatStore.chatData.length
    };
    winStore.loadingStream = true;

    // fetch stream
    if (chatStore.aiModel == "azure" || chatStore.aiModel == 'gpt4') {
      chatStore.typing = true;
      streamChat(sendData);
    } else if (chatStore.aiModel == 'embed') {
      chatStore.typing = true;
      chatStore.pdfMetaData = [];
      streamChat({...sendData, embeddingId: chatStore.embeddingId});
    }
    // rest api
    else {
      instance
        .post("/chat", sendData)
        .then((res) => {
          streaming.value = false;
          let apiResponse = res.data.result;
          if (chatStore.aiModel == 'bard' || chatStore.aiModel == 'llama2') {
            let answerText : string = apiResponse.answer;

            chatStore.typing = true;
            chatStore.chatData[chatStore.chatData.length - 1].content =
              answerText;
          } else if (chatStore.aiModel == 'image') {
            if (apiResponse.image_list[0] == 'IMAGE_GENERATION_FAILED') {
              chatStore.chatData[chatStore.chatData.length - 1].content = 
              'Image creation failed. Please try again with a different keyword. (English only)';
              chatStore.chatData[chatStore.chatData.length - 1].imageList = [];
            } else {
              chatStore.typing = true;
              chatStore.chatData[chatStore.chatData.length - 1].content = '';
              chatStore.chatData[chatStore.chatData.length - 1].imageList = 
                apiResponse.image_list;
            }

          }
        })
        .catch((err) => {
          console.log(err);
          streaming.value = false;
          winStore.loadingStream = false;
        });
    }
  }
  if (askText.value.trim() !== "") {
    router.push(
      `/gen-ai/c/code${new Date().getTime()}`
    );
  }

  const textarea = textareaRef.value;
  if (!textarea) {
    return;
  }

  textarea.style.height = "24px";
  askText.value = "";
};
let trueVal = true;

let controller = new AbortController();

const streamChat = async (sendData: any) => {
  controller = new AbortController();
  trueVal = true;
  try {
    const response = await fetch(
      `${import.meta.env.VITE_API_URL}/sse/azure`,
      {
        method: "POST",
        body: JSON.stringify(sendData),
        headers: {
          "Content-Type": "application/json",
          Authorization: `Bearer ${sessionStorage.getItem("access_token")}`,
        },
        signal: controller.signal,
      }
    )
    
    if (!response.ok) {
      chatStore.chatData[chatStore.chatData.length - 1].content =
      '죄송합니다. 현재 시스템에 오류가 발생하여 답변을 드릴 수 없습니다.';

      chatStore.chatData[chatStore.chatData.length - 1].isError = true;

      trueVal = false;
      winStore.loadingStream = false;
      chatStore.typing = false;
      streaming.value = false;
      regenYn.value = 'N';
    } else {

      const reader = response.body!.getReader();
      const decoder = new TextDecoder();
      
      let answerText = "";
      let truncate = "";

      while (trueVal) {
        const { done, value } = await reader.read();
        let chunk = decoder.decode(value);

        chunk = truncate + chunk;
        let index = chunk.lastIndexOf("\n\n");
        
        let parseChunk = chunk.substring(0, index);
        truncate = chunk.substring(index+1, chunk.length);

        const lines = parseChunk.split("\n\n");
        
        let parsedLines = lines
        .map((line) => line.trim().replace(/^data:/, ""))
        .filter((line) => line !== "" && line !== "[DONE]")
        .map((line) => {
          return JSON.parse(line);
        });

        for (const parsedLine of parsedLines) {
          const { choices } = parsedLine;
          const { delta } = choices[0];
          const content = delta.content;
          
          if(choices[0].finish_reason == 'stop'){
            trueVal = false;
            winStore.loadingStream = false;
            chatStore.typing = false;
            
              if (chatStore.chatData.length == 2 && regenYn.value == 'N') {
                const sessionId = sessionStorage.getItem("sessionId");
                chatStore.sessionId = sessionId == null ? "" : sessionId;
                if (chatStore.chatTitleData.length == 0) {
                  chatStore.chatTitleData.unshift({
                    chatGroup: 0,
                    titleList: [
                      {
                        title: choices[0].summary_title,
                        sessionId: chatStore.sessionId,
                        aiModel: chatStore.aiModel,
                        embeddingId: chatStore.embeddingId,
                        chatTime: new Date().getTime(),
                      },
                    ],
                  });
                } else if (
                  chatStore.chatTitleData.length !== 0 &&
                  chatStore.chatTitleData[0].chatGroup !== 0
                ) {
                  chatStore.chatTitleData.unshift({
                    chatGroup: 0,
                    titleList: [
                      {
                        title: choices[0].summary_title,
                        sessionId: chatStore.sessionId,
                        aiModel: chatStore.aiModel,
                        embeddingId: chatStore.embeddingId,
                        chatTime: new Date().getTime(),
                      },
                    ],
                  });
                } else {
                  chatStore.chatTitleData[0].titleList.unshift({
                    title: choices[0].summary_title,
                    sessionId: chatStore.sessionId,
                    aiModel: chatStore.aiModel,
                    embeddingId: chatStore.embeddingId,
                    chatTime: new Date().getTime(),
                  });
                }
              }
            streaming.value = false;
            regenYn.value = 'N';
          } else if (choices[0].finish_reason == 'error') {
            chatStore.chatData[chatStore.chatData.length - 1].isError = true;
          }

          if (content) {
            answerText += content;
          }

        }

        if (done) {
          trueVal = false;
          winStore.loadingStream = false;
          chatStore.typing = false;
          streaming.value = false;
          regenYn.value = 'N';
          
          break;
        }

        if (answerText.length > 0) {
          const intervalId = setInterval(() => {
            let char = answerText.slice(0, 1);
            answerText = answerText.slice(1);
            chatStore.chatData[chatStore.chatData.length - 1].content += char;
            if (answerText.length == 0) {
              clearInterval(intervalId);
            }
          }, 80);
        }
      }

      if (chatStore.aiModel == 'embed' && 
        !chatStore.chatData[chatStore.chatData.length - 1].isError
      ) {
        emptySrc();

        getPdfPage().then((ress) => {
          ress.pdf_metadata.map((item : {pdf : string, pageIndex : number}) => {
            chatStore.pdfMetaData.push({
              pdfSrc : item.pdf,
              pdfPage : item.pageIndex
            })
          });
          let responseSrc = ress.pdf_metadata[0].pdf;
          chatStore.selectedPdf = formatter.splitFileName(responseSrc);
          chatStore.selectedRefer = 0;
          chatStore.pdfSrc = '/img/gen/' + chatStore.pdfMetaData[0].pdfSrc.substring(chatStore.pdfMetaData[0].pdfSrc.search('genai'));
          chatStore.pdfPage = ress.pdf_metadata[0].pageIndex + 1;

          winStore.loading = false;
        });
      }
    }
    
  } catch (err) {
    if (userSvc.getMode() == 'ADMIN') {
      console.log('에러발생@@@', err);
    }
    return;
  }
};

const emptySrc = async () => {
  return chatStore.pdfSrc = '';
}

const handleRegen = (e : Event) => {
  
  regenYn.value = 'Y';
  askText.value = chatStore.chatData[chatStore.chatData.length - 2].content;
  trueVal = true;
  chatStore.chatData.splice(-2);
  askSend(e);
}

const handleStop = () => {
  trueVal = false;
  controller.abort();

  const sessionId = sessionStorage.getItem("sessionId");
  chatStore.sessionId = sessionId ?? '';
  winStore.loadingStream = false;
  chatStore.typing = false;
  streaming.value = false;
}

</script>

<style lang="scss" scoped>
button,
input,
textarea {
  color: inherit;
  font-family: inherit;
  font-size: 100%;
  font-weight: inherit;
  line-height: inherit;
  margin: 0;
}

textarea {
  appearance: none;
  border-color: #8e8ea0;
  border-radius: 0;
  border-width: 1px;
  font-size: 1rem;
  line-height: 1.5rem;

  &:focus {
    border-color: inherit;
    outline: none;
  }
  &::placeholder {
    color: #9b9ea3;
  }
}

.btn-area {
  .btn {
    align-items: center;
    border-color: transparent;
    border-radius: 0.25rem;
    border-width: 1px;
    display: inline-flex;
    font-size: 0.875rem;
    line-height: 1.25rem;
    padding: 0.5rem 0.75rem;
    pointer-events: auto;
    appearance: button;
    background-image: none;
    cursor: pointer;
    text-transform: none;
  }
}

.btn-neutral {
  --tw-bg-opacity: 1;
  --tw-text-opacity: 1;
  background-color: rgba(255, 255, 255, var(--tw-bg-opacity));
  border-color: rgba(0, 0, 0, 0.1);
  border-width: 1px;
  color: rgba(64, 65, 79, var(--tw-text-opacity));
  font-size: 0.875rem;
  line-height: 1.25rem;

  &:hover {
    --tw-bg-opacity: 1;
    background-color: rgba(236, 236, 241, var(--tw-bg-opacity));
  }
}

.btn {
  align-items: center;
  border-color: transparent;
  border-radius: 0.25rem;
  border-width: 1px;
  display: inline-flex;
  font-size: 0.875rem;
  line-height: 1.25rem;
  padding: 0.5rem 0.75rem;
  pointer-events: auto;
}
</style>
