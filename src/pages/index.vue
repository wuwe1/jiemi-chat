<script setup lang="ts">
let isPending = $ref(false)
let question = $ref('')

const prompts = {
  'let\'s think step by step': () =>
    question += ' let\'s think step by step',
  'dictionary': () =>
    question += 'what is the phonetic transcript of "word" without other text and what is the meaning of the word',
}

interface Usage {
  prompt_tokens: number
  completion_tokens: number
  total_tokens: number
}
interface Message {
  content: string
  usage?: Usage
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
  const response = await fetch('https://api.jiemichat.wuwe1.com:8000', {
    method: 'POST',
    body: JSON.stringify(data),
  })
  const { content, usage } = await response.json() as Message
  if (response.ok) {
    qaArray.push({
      answer: { content, usage },
      question: { content: question },
    })
  }

  isPending = false
  question = ''
}

const onKeyDown = (e: KeyboardEvent) => {
  if (e.ctrlKey || e.metaKey) {
    if (e.key === 'Enter')
      go()
  }
}
</script>

<template>
  <div class="container" grid grid-cols-8>
    <nav col-span-2 h-full border-x border-gray-2 dark:border-gray-7 flex="~ col" justify-between>
      <div>
        <div border-b border-gray-2 dark:border-gray-7 py-2 uppercase font-mono>
          <div py-2>
            <img src="../../public/wt.jpeg" alt="" w-20 rounded-full m-auto>
          </div>
          Jiemi Chat
        </div>
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
      <div border-t border-gray-2 dark:border-gray-7 py-2>
        <TheFooter />
      </div>
    </nav>
    <main
      col-span-6
      relative
      border-r border-gray-2 dark:border-gray-7
    >
      <article
        v-if="qaArray.length === 0"
        h-screen flex items-center justify-center
      >
        <p>Welcome to JieMi Chat</p>
      </article>
      <article v-else text-left px-8 h-screen overflow-scroll pb-30>
        <div v-for="(qa, index) in qaArray" :key="index">
          <div font-mono text-green-3>
            {{ qa.question.content }}
          </div>
          <div p="x-4 y-2" dark:text-gray-2 border="~ rounded none" v-html="qa.answer.content" />
          <div text-right bg-blue-2 dark:bg-blue-7 px-2 text-sm>
            total tokens: {{ qa.answer.usage?.total_tokens }} / 4096
          </div>
          <br>
        </div>
      </article>

      <div w-full h-32 absolute left-0 bottom-0>
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

.container {
  height: 100vh;
}

@media screen and (max-width: 768px) {
  .container {
    height: calc(100vh - 56px);
  }
}
</style>
