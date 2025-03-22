<template>
  <div class="flex-1 overflow-hidden">
    <div class="relative h-full">
      <div class="scrollbox react-scroll-to-bottom" ref="chatScroll">
        <div class="flex flex-col text-sm">

          <div v-for="(chat, index) in chatStore.chatData" :key="index">
            <gen-ask-message v-if="chat.role == 'user' && chat.sessionId == nowSessionId">
              {{ chat.content }}
            </gen-ask-message>
            <gen-answer-message :aiModel="chatStore.aiModel" v-else-if="chatStore.typing == true && index == chatStore.chatData.length - 1">
              <div
                class="markdown prose w-full break-words dark:prose-invert light"
                v-html="markdown.render(displayResponse)"
              >
              </div>
            </gen-answer-message>
            <gen-answer-message :aiModel="chatStore.aiModel" v-else-if="chatStore.aiModel == 'image' && chat.sessionId == nowSessionId">
              <span v-if="chat.content">
                {{ chat.content }}
              </span>
              <n-carousel
                v-else
                :show-arrow="chat.imageList!.length > 3"
                :slides-per-view="3"
                :space-between="20"
                :loop="false"
              >
                <div v-for="(item, index) in chat.imageList" :key="index">
                  <img
                    class="carousel-img cursor-pointer"
                    :src="'/img/gen/' + item.substring(item.search('gen_image'))"
                    @click.stop="showBigImg(item)"
                  />
                </div>
                <template #dots="{ total, currentIndex, to }">
                  <ul class="custom-dots">
                    <li
                      v-for="n of ((total+2)%3==0?Math.trunc((total+2)/3) : Math.ceil((total+2)/3))"
                      :key="n"
                      :class="{['is-active']: currentIndex === ((n-1)*3 >= total ? total-1 : (n-1)*3)}"
                      @click="to((n-1)*3 > total ? total-1 : (n-1)*3)"
                    >
                    </li>
                  </ul>
                </template>
                <template #arrow="{ prev, next }">
                  <div class="custom-arrow pr-2 gap-3">
                    <button type="button" class="custom-arrow-left"
                      @click="() => { prev() }"
                    >
                      ◀
                    </button>
                    <button type="button" class="custom-arrow-right"
                      @click="() => { next() }"
                    >
                      ▶
                    </button>
                  </div>
                </template>
              </n-carousel>
            </gen-answer-message>
            <gen-answer-message v-else :aiModel="chatStore.aiModel">
              <div
                class="markdown prose w-full break-words dark:prose-invert light"
                v-html="markdown.render(chat.content)"
                @click="copyCode"
              >
              </div>
              <span
                v-if="chatStore.aiModel == 'embed' && chatStore.chatData.length - 1 == index"
                class="flex gap-2"
              >
                <span
                  v-for="(item, idx) in chatStore.pdfMetaData"
                  :key="idx"
                  class="pdfRefer"
                  :class="{ 'selected-refer' : idx == chatStore.selectedRefer }"
                  @click="handleChangePdf(item, idx)"
                >
                  <n-tooltip class="text-xs" placement="bottom" trigger="hover">
                    <template #trigger>
                      참조{{ idx + 1 }}&nbsp;
                    </template>
                    {{ toEllipsis(formatter.splitFileName(item.pdfSrc)) }}<br>
                    Page: {{ item.pdfPage + 1 }}
                  </n-tooltip>
                </span>
              </span>
              <div class="flex justify-between lg:block"
                v-if="chatStore.aiModel != 'image'"
              >
                <div
                  class="cmd-buttons flex self-end lg:self-center justify-center mt-2 gap-2 md:gap-3 lg:gap-1 lg:absolute lg:top-0 lg:translate-x-full lg:right-0 lg:mt-0 lg:pl-2 visible"
                >
                <n-tooltip class="text-tooltip" trigger="hover">
                  <template #trigger>
                    <button
                      class="rounded-md"
                      v-if="isCopy && index == contentIdx"
                    >
                      <check-icon />
                    </button>
                    <button
                      class="p-1 rounded-md hover:bg-gray-100 hover:text-gray-700"
                      @click="copyContent(index)"
                      v-else
                    >
                      <copy-icon />
                    </button>
                  </template>
                  Copy
                </n-tooltip>
                </div>
              </div>
            </gen-answer-message>
          </div>
          <div class="h-32 md:h-48 flex-shrink-0"></div>
        </div>
      </div>
      <v-dialog
        v-model="isBig"
        max-width="50rem"
        max-height="50rem"
        persistent
        :no-click-animation="true"
      >
        <v-card
          class="bigImgContainer"
        >
          <button
            class="absolute left-0 top-0"
            @click="() => isBig = false"
          >
            <close-icon class="closeIcon"/>
          </button>
          <img class="bigImg cursor-pointer"
          :src="'/img/gen/' + imgSrc.substring(imgSrc.search('gen_image'))"
          @click="() => isBig = false"
          />
        </v-card>
      </v-dialog>
    </div>
  </div>
</template>

<script setup lang="ts">
import { useGenAiChatDataStore, useGenAiModelsStatusStore, useWindowStatusStore } from "@/store";
import { onBeforeUnmount, onMounted, onUpdated, ref, watchEffect } from "vue";
import { getPdfPage, getEmbedInfo } from "@/api/chat";
import { NCarousel, NTooltip } from "naive-ui";
import { instance } from "@/api";
import router from "@/router";
import MarkdownIt from "markdown-it";
import MarkdownItMultimdTable from 'markdown-it-multimd-table';
import { formatter } from "@/utils";
import hljs from 'highlight.js';
import 'highlight.js/styles/stackoverflow-dark.css';

const chatStore = useGenAiChatDataStore();
const aiModelStore = useGenAiModelsStatusStore();
const winStore = useWindowStatusStore();

const chatScroll = ref<HTMLDivElement | null>(null);
const isBottom = ref(true);
const scrollToBottom = () => {
  if(chatScroll.value && isBottom.value) {
    chatScroll.value.scrollTop = chatScroll.value.scrollHeight;
  }
}

const handleScroll = () => {
  if (chatScroll.value) {
    const scrollPosition = chatScroll.value.scrollTop;
    const scrollHeight = chatScroll.value.scrollHeight;
    const clientHeight = chatScroll.value.clientHeight;

    isBottom.value = scrollPosition + clientHeight + 30 >= scrollHeight;
  }
};

const displayResponse = ref('');
const startTypingEffect = () => {
  if((chatStore.aiModel == 'bard' || chatStore.aiModel == 'llama2' || chatStore.aiModel == 'image') && chatStore.typing){
    let i = 0;
    displayResponse.value = '';
    const responseData = chatStore.chatData[chatStore.chatData.length - 1]?.content;
    const intervalId = setInterval(() => {
      displayResponse.value = responseData.slice(0, i);
      i++;
      if (i > responseData.length) {
        clearInterval(intervalId);
        chatStore.typing = false;
      }
    }, 20);
  } else {
    const stringResponse = chatStore.chatData[chatStore.chatData.length - 1]?.content;
    displayResponse.value = stringResponse;
    scrollToBottom();
  }
};

watchEffect(() => {
  startTypingEffect();
});

let nowSessionId = sessionStorage.getItem('sessionId');

onMounted(() => {
  if (!chatStore.sessionId) {
    chatStore.chatData = [];
    chatStore.sessionId = nowSessionId;
    chatStore.embeddingId = sessionStorage.getItem('embeddingId');

    instance.get(`/chat/${nowSessionId}`)
      .then(res => {
        if (!res.data.length) {
          router.push('/gen-ai/new');
          return;
        }

        chatStore.aiModel = res.data[0].chat_type;

        if (chatStore.aiModel !== 'image') {
          res.data.forEach(
            (item: {
              question: string,
              result: { answer: string; role: string; embedding_id: string }
            }) => {
              chatStore.chatData.push(
                {
                  role: "user",
                  content: item.question,
                  sessionId: nowSessionId,
                  embeddingId: item.result.embedding_id,
                },
                {
                  role: "assistant",
                  content: item.result.answer,
                  sessionId: nowSessionId,
                  embeddingId: item.result.embedding_id,
                }
              );
              sessionStorage.setItem("embeddingId", item.result.embedding_id);
            }
          );
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
                  sessionId: nowSessionId,
                  embeddingId: null,
                },
                {
                  role: "assistant",
                  content: item.result.image_list[0] == 'IMAGE_GENERATION_FAILED'
                    ? '이미지 생성을 실패하였습니다. 다른 키워드로 재시도 해주세요.'
                    : '',
                  imageList: item.result.image_list[0] == 'IMAGE_GENERATION_FAILED'
                    ? []
                    : item.result.image_list,
                  sessionId: nowSessionId,
                  embeddingId: null,
                }
              );
            }
          );
        }
        
        if (chatStore.chatData[0].embeddingId && chatStore.chatData[0].embeddingId != 'null') {
          chatStore.aiModel = 'embed';
          aiModelStore.isEmbed = true;

          getPdfPage().then((ress) => {
            ress.pdf_metadata.map((item: { pdf: string, pageIndex: number }) => {
              chatStore.pdfMetaData.push({
                pdfSrc: item.pdf,
                pdfPage: item.pageIndex
              })
            })

            let responseSrc = ress.pdf_metadata[0].pdf;
            chatStore.selectedPdf = formatter.splitFileName(responseSrc);
            chatStore.selectedRefer = 0;
          });

          getEmbedInfo(chatStore.chatData[0].embeddingId).then(ress => {
            chatStore.setPdfState(1);
            chatStore.setImageData(ress.images);
            let responseSrc = ress.pdf_metadata[0].pdf;

            chatStore.pdfSrc = '/img/gen/' + responseSrc.substring(responseSrc.search('genai'));
            winStore.loading = false;
          })
        } else {
          aiModelStore.isEmbed = false;
        }

        chatScroll.value?.addEventListener('scroll', handleScroll);
      })
      .catch((err) => {
        console.log(err);
      });
  }
  
});

onBeforeUnmount(() => {
  chatScroll.value?.removeEventListener('scroll', handleScroll);
});

onUpdated(() => {
  nowSessionId = sessionStorage.getItem('sessionId');
  
  chatScroll.value?.addEventListener('scroll', handleScroll);
  scrollToBottom();
})

const isBig = ref(false);
const imgSrc = ref('');
const showBigImg = (src : string) => {
  isBig.value = true;
  imgSrc.value = src;
};

const isCopy = ref(false);
const contentIdx = ref(0);
const copyContent = (idx : number) => {
  contentIdx.value = idx;
  let content = chatStore.chatData[idx].content;

  navigator.clipboard.writeText(content)
    .then(() => {
      isCopy.value = true;

      setTimeout(() => {
        isCopy.value = false;
      }, 2000);
    })
    .catch(err => {
      console.log(err);
      chatStore.alertContent = `에러가 발생하였습니다.\n관리자에게 문의해주세요.`;
      chatStore.openAlert = true;
    })
}

const copyCode = (e : any) => {
  if (e.target.classList.contains('code-copy-btn')) {
    const targetCodeBlock = e.target.parentElement.parentElement.children[1].children[0].textContent;

    navigator.clipboard.writeText(targetCodeBlock)
    .then(() => {
      e.target.textContent = 'Copied!';
      e.target.className = 'copied-btn';
      setTimeout(() => {
        e.target.textContent = 'Copy code';
        e.target.className = 'code-copy-btn';
      }, 2000);
    })
  }
}

const markdown = new MarkdownIt({
  highlight: function (str: string, lang: string) {

    function createCodeBlock() {

      const preElement = document.createElement('pre');

      const mainDiv = document.createElement('div');
      mainDiv.className = 'bg-black rounded-md';

      const headerDiv = document.createElement('div');
      headerDiv.className = 'flex items-center bg-gray-800 text-xs justify-between px-4 py-2 rounded-t-md font-code';

      const langSpan = document.createElement('span');
      langSpan.textContent = lang;

      const copyButton = document.createElement('button');
      copyButton.type = 'button';
      copyButton.textContent = 'Copy code';
      copyButton.className = 'code-copy-btn';

      headerDiv.appendChild(langSpan);
      headerDiv.appendChild(copyButton);

      const contentDiv = document.createElement('div');
      contentDiv.className = 'overflow-y-auto';

      const codeElement = document.createElement('code');
      codeElement.className = `!whitespace-pre hljs language-${lang} code-scrollbox`;
      codeElement.innerHTML = hljs.highlight(str, {language: lang, ignoreIllegals: true}).value;

      contentDiv.appendChild(codeElement);
      mainDiv.appendChild(headerDiv);
      mainDiv.appendChild(contentDiv);
      preElement.appendChild(mainDiv);

      return preElement.outerHTML;
    }
    
    function noLangCodeBlock() {
      const preElement = document.createElement('pre');

      const mainDiv = document.createElement('div');
      mainDiv.className = 'bg-black rounded-md';

      const headerDiv = document.createElement('div');
      headerDiv.className = 'flex items-center bg-gray-800 text-xs justify-between px-4 py-2 rounded-t-md font-code';

      const langSpan = document.createElement('span');
      langSpan.textContent = lang;

      const copyButton = document.createElement('button');
      copyButton.type = 'button';
      copyButton.className = 'code-copy-btn';
      copyButton.textContent = 'Copy code';
      
      headerDiv.appendChild(langSpan);
      headerDiv.appendChild(copyButton);

      const contentDiv = document.createElement('div');
      contentDiv.className = 'overflow-y-auto';

      const codeElement = document.createElement('code');
      codeElement.className = `!whitespace-pre hljs code-scrollbox`;
      codeElement.innerHTML = hljs.highlightAuto(str).value;

      contentDiv.appendChild(codeElement);
      mainDiv.appendChild(headerDiv);
      mainDiv.appendChild(contentDiv);
      preElement.appendChild(mainDiv);

      return preElement.outerHTML;
    }
    if (lang && hljs.getLanguage(lang)) {
      try {
        return createCodeBlock();
      } catch (err) {
        console.log(err);
      }
    }

    return noLangCodeBlock();
  }
}).use(MarkdownItMultimdTable);

const handleChangePdf = async (data : {pdfSrc : string, pdfPage : number}, idx : number) => {  
  await emptySrc();
  
  chatStore.selectedRefer = idx;
  chatStore.selectedPdf = formatter.splitFileName(data.pdfSrc);
  chatStore.pdfSrc = '/img/gen/' + data.pdfSrc.substring(data.pdfSrc.search('genai'));
  chatStore.pdfPage = data.pdfPage + 1;
  
}

const emptySrc = async () => {
  return chatStore.pdfSrc = '';
}

const toEllipsis = (text : string) => {
  return text.length > 26 ? text.substring(0, 26) + '...' : text;
}
</script>

<style scoped lang="scss">

.cursor {
  display: inline-block;
  width: 1ch;
  animation: flicker 0.5s infinite;
  margin-bottom: 4px;
}
@keyframes flicker {
  0% {
    opacity: 0;
  }
  50% {
    opacity: 1;
  }
  100% {
    opacity: 0;
  }
}

.cmd-buttons {
  color: #818590;
}
.-translate-x-full,
.-translate-y-1\/2 {
  --tw-translate-x: 0;
  --tw-translate-y: 0;
  --tw-rotate: 0;
  --tw-skew-x: 0;
  --tw-skew-y: 0;
  --tw-scale-x: 1;
  --tw-scale-y: 1;
  -webkit-transform: translate(var(--tw-translate-x), var(--tw-translate-y))
    rotate(var(--tw-rotate)) skewX(var(--tw-skew-x)) skewY(var(--tw-skew-y))
    scaleX(var(--tw-scale-x)) scaleY(var(--tw-scale-y));
  transform: translate(var(--tw-translate-x), var(--tw-translate-y))
    rotate(var(--tw-rotate)) skewX(var(--tw-skew-x)) skewY(var(--tw-skew-y))
    scaleX(var(--tw-scale-x)) scaleY(var(--tw-scale-y));
}
.-translate-x-full {
  --tw-translate-x: -100%;
}
.hover\:text-gray-700:hover {
  --tw-text-opacity: 1;
  color: rgba(64, 65, 79, var(--tw-text-opacity));
}
.hover\:bg-gray-100:hover {
  --tw-bg-opacity: 1;
  background-color: rgba(236, 236, 241, var(--tw-bg-opacity));
}

.carousel-img {
  width: 12rem;
  height: 12rem;
  object-fit: cover;
}

.custom-arrow {
  display: flex;
  position: absolute;
  bottom: 1rem;
  right: 1rem;
}

.custom-arrow button {
  align-items: center;
  justify-content: center;
  width: 1.8rem;
  height: 1.8rem;
  color: #fff;
  background-color: rgba(104, 99, 99, 0.3);
  border-width: 0;
  border-radius: 50%;
  transition: background-color 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  cursor: pointer;
}

.custom-arrow button:hover {
  background-color: rgba(206, 206, 206, 0.5);
}

.custom-arrow button:active {
  transform: scale(0.9);
  transform-origin: center;
}

.custom-dots {
  display: flex;
  margin: 0;
  padding: 0;
  position: absolute;
  bottom: 1rem;
  left: 1rem;
}

.custom-dots li {
  display: inline-block;
  width: 1rem;
  height: 0.5rem;
  margin: 0 3px;
  border-radius: 4px;
  background-color: rgba(255, 255, 255, 0.7);
  /* background-color: red; */
  transition: width 0.3s, background-color 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  cursor: pointer;
}

.custom-dots li.is-active {
  width: 2rem;
  background: #fff;
  /* background-color: red; */
}

.bigImgContainer {
  width: 100%;
  height: 100%;
  padding: 1rem;
}

.bigImg {
  width: 100%;
  height: 100%;
}

.pdfRefer {
  cursor: pointer;
  text-decoration: underline;
  color: rgba(0, 0, 255, 0.4);
  font-style: italic;
  font-weight: bold;

    &:hover {
      color: blue;
    }
}

.selected-refer {
  color: blue;
}

.closeIcon {
  width: 3rem;
  height: 3rem;
  color: #65686e;
}

</style>
