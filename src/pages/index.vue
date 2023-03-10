<script setup lang="ts">
import MarkdownIt from 'markdown-it'
import hljs from 'highlight.js'

interface Usage {
  prompt_tokens: number
  completion_tokens: number
  total_tokens: number
}
interface Message {
  content: string
  usage?: Usage
  timeElapsed?: number
}

interface QA {
  question: Message
  answer: Message
}

const qaArray = reactive<QA[]>([])

const clear = () => {
  const arr = unref(qaArray)
  arr.splice(0, arr.length)
}

const speechText = ref('')
const voice = ref<SpeechSynthesisVoice>(undefined as unknown as SpeechSynthesisVoice)
const playingIndex = ref(undefined as unknown as number)
const voices = ref<SpeechSynthesisVoice[]>([])
const speech = useSpeechSynthesis(speechText, { voice })

onMounted(() => {
  if (speech.isSupported.value) {
    setTimeout(() => {
      voices.value = window.speechSynthesis.getVoices()
      voice.value = voices.value[0]
    })
  }
})

const isPlaying = (index: number) => {
  return index === playingIndex.value && speech.status.value === 'play'
}

const play = (index: number) => {
  if (playingIndex.value !== index) {
    playingIndex.value = index
    window.speechSynthesis.cancel()
    speech.toggle(false)
    speechText.value = qaArray[index].answer.content
    speech.speak()
    return
  }

  if (speech.status.value === 'pause')
    window.speechSynthesis.resume()
  else
    speech.speak()
}

const pause = () => {
  window.speechSynthesis.pause()
}

let isPending = $ref(false)
let question = $ref('')

const md = new MarkdownIt({
  highlight(str: string) {
    try {
      return hljs.highlightAuto(str).value
    }
    catch (__) {}

    return ''
  },
})

const API_ENDPOINT = 'https://chat-api.wuwe1.workers.dev'

const prompts = {
  'step by step': () =>
    question += ' let\'s think step by step',
  'dict': () =>
    question += 'what is the phonetic transcript of "word" without other text and what is the meaning of the word',
}

const go = async () => {
  if (question.length === 0)
    return
  const currentQuestion = {
    content: question,
    role: 'user',
  }
  const data = [...qaArray.values()].map(({ question, answer }) => {
    return [{
      role: 'user',
      content: question.content,
    },
    {
      role: 'assistant',
      content: answer.content,
    }]
  }).reduce((prev, cur) => prev.concat(cur), []).concat(currentQuestion)

  isPending = true
  try {
    const start = (new Date()).getTime()
    const response = await fetch(API_ENDPOINT, {
      method: 'POST',
      body: JSON.stringify(data),
    })
    const { content, usage } = await response.json() as Message
    if (!response.ok)
      throw new Error('Api Error')
    qaArray.push({
      answer: { content, usage, timeElapsed: (new Date()).getTime() - start },
      question: { content: question },
    })
    question = ''
  }
  catch (e) {

  }
  finally {
    isPending = false
  }
}

const onKeyDown = (e: KeyboardEvent) => {
  if (e.ctrlKey || e.metaKey) {
    if (e.key === 'Enter')
      go()
  }
}
</script>

<template>
  <div class="fit" grid grid-cols-9>
    <nav col-span-1 h-full border-x border-gray-2 dark:border-gray-7>
      <div>
        <div pt-2>
          <button
            v-for="(v, k) in prompts" :key="k"
            m-1 px-2
            text-white
            bg-blue-3 dark:bg-blue-6 rounded text-sm uppercase font-mono
            hover:bg-blue-2 dark:hover:bg-blue-5
            @click="v"
          >
            {{ k }}
          </button>
        </div>
      </div>
    </nav>
    <main
      col-span-8 flex="~ col" relative
      border-r border-gray-2 dark:border-gray-7 overflow-hidden
    >
      <div
        absolute top-0 w-full
        backdrop-blur
        flex justify-between items-center
        border-b border-gray-2 dark:border-gray-7 p-2 uppercase font-mono
      >
        <div>
          <img inline-block src="../../public/wt.jpeg" alt="" w-10 rounded-full m-auto>
          <span text-lg> JieMi Chat </span>
        </div>
        <TheFooter />
      </div>
      <article
        v-if="qaArray.length === 0"
        h-full flex items-center justify-center
      >
        <div>
          <p>👋 Welcome to JieMi Chat!</p>
          <p>press <code>⌘ + enter</code> or <code>ctrl + enter</code> to submit</p>
          <p>press the left hand side buttom use predefined prompt</p>
        </div>
      </article>
      <article v-else text-left px-8 overflow-y-auto flex-1 pt-16>
        <div v-for="(qa, index) in qaArray" :key="index">
          <div font-mono text-green-3>
            {{ qa.question.content }}
          </div>
          <div p="x-4 y-2" dark:text-gray-2 border="~ rounded none" v-html="md.render(qa.answer.content)" />
          <div v-if="speech.isSupported">
            <button v-if="isPlaying(index)" icon-btn i-carbon-stop-filled bg-green-4 dark:bg-green-2 @click="pause" />
            <button v-else icon-btn i-carbon-play-filled bg-green-4 dark:bg-green-2 @click="() => play(index)" />
          </div>
          <div text-right bg-blue-2 dark:bg-blue-7 px-2 text-sm>
            {{ `took ${qa.answer.timeElapsed! / 1000}s total tokens: ${qa.answer.usage?.total_tokens}/4096` }}
          </div>
          <br>
        </div>
      </article>

      <div w-full h-32>
        <div w-full px-6 pb-2>
          <div h-20 flex items-center>
            <div
              v-if="isPending"
              rounded w-full h-2
              class="gradient-bg-animation"
            />
            <TheTextArea
              v-else
              v-model="question"
              w-full
              font-mono :disabled="isPending"
              @keydown="onKeyDown"
            />
          </div>
          <div flex flex-gap-2>
            <button text-sm btn w-full :disabled="isPending" @click="go">
              Go
            </button>
            <button text-sm btn w-full :disabled="isPending" @click="clear">
              Clear
            </button>
          </div>
        </div>
      </div>
    </main>
  </div>
</template>

<style>
.gradient-bg-animation {
  height: 4px;
  width: 18rem;
  border: transparent;
  background: linear-gradient(-45deg, #3dd433, #3f87a6, #f69d3c, #e73c7e);
  background-size: 400% 400%;
  animation: gradient 3s ease infinite;
}

@keyframes gradient {
  0% {
    background-position: 0% 50%;
  }

  50% {
    background-position: 100% 50%;
  }

  100% {
    background-position: 0% 50%;
  }
}

.fit {
  height: 100vh;
}

@media screen and (max-width: 768px) {
  .fit {
    height: calc(100vh - 56px);
  }
}
</style>
