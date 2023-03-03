<script setup lang="ts">
let isPending = $ref(false)
let question = $ref('')

const prompts = {
  'let\'s': () => {
    question += ' let\'s think step by step'
  },
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
  for (let i = 0; i < qaArray.length; i++)
    qaArray.pop()
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
  const response = await fetch('http://159.223.88.132:8000', {
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
  <div h-screen flex>
    <nav w-64 h-screen flex="~ col" justify-between>
      <div>
        <div border-b border-gray-7 py-2 uppercase font-mono>
          Jiemi Chat
        </div>
        <div pt-2>
          <button
            v-for="(v, k) in prompts" :key="k"
            bg-blue-3 dark:bg-blue-6 px-2 rounded text-lg uppercase font-mono
            hover:bg-blue-2 dark:hover:bg-blue-5
            @click="v"
          >
            {{ k }}
          </button>
          <div py-2>
            <img src="../../public/wt.jpeg" alt="" w-20 rounded-full m-auto>
          </div>
        </div>
      </div>
      <div border-t border-gray-7 py-2>
        <TheFooter />
      </div>
    </nav>
    <div border-x border-gray-7 w-screen-md h-screen overflow-scroll>
      <div text-left px-8>
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
      </div>

      <div h-screen />

      <div
        pt-2
        border-t border-gray-7
        sticky bottom-0
        bg-white dark:bg-gray-9
      >
        <div v-if="isPending" m-auto class="gradient-bg-animation" h-2 />
        <TheTextArea
          v-else
          v-model="question"
          min-w-sm max-w-lg font-mono :disabled="isPending"
          @keydown="onKeyDown"
        />
        <div>
          <button class="m-3 text-sm btn" :disabled="isPending" @click="go">
            Go
          </button>
          <button class="m-3 text-sm btn" :disabled="isPending" @click="clear">
            Clear
          </button>
        </div>
      </div>
    </div>
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
</style>
