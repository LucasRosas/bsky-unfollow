<script setup>
import {
  Dialog,
  DialogContent,
  DialogDescription,
  DialogHeader,
  DialogTitle,
} from '@/components/ui/dialog'

import {
  TagsInput,
  TagsInputInput,
  TagsInputItem,
  TagsInputItemDelete,
  TagsInputItemText,
} from '@/components/ui/tags-input'
import {
  NumberField,
  NumberFieldContent,
  NumberFieldDecrement,
  NumberFieldIncrement,
  NumberFieldInput,
} from '@/components/ui/number-field'
import { Button } from '@/components/ui/button'
import Card from './ui/card/Card.vue'
import IconCalendarX from './icons/IconCalendarX.vue'
import IconFunnelSimpleX from './icons/IconFunnelSimpleX.vue'
import Switch from './ui/switch/Switch.vue'
import IconUserCircleMinus from './icons/IconUserCircleMinus.vue'
import { ref, computed, watch } from 'vue'
import IconThumbsDown from './icons/IconThumbsDown.vue'

const props = defineProps({
  openConfig: {
    type: Boolean,
  },
  config: {
    type: Object,
    default: () => {
      return {
        countDays: 30,
        filterByDays: false,
        filterByMoots: false,
        filterByWords: false,
        filterBySpam: false,
        keyWords: [],
        mode: 'list',
      }
    },
  },
})

const filterByDays = ref(false)
const countDays = ref(30)
const filterByWords = ref(false)
const filterByMoots = ref(false)
const filterBySpam = ref(false)
const keyWords = ref([])
const mode = ref('list')

watch(
  () => props.config,
  (value) => {
    filterByDays.value = value.filterByDays
    countDays.value = value.countDays
    filterByWords.value = value.filterByWords
    keyWords.value = value.keyWords
    filterByMoots.value = value.filterByMoots
    filterBySpam.value = value.filterBySpam
    mode.value = value.mode
  },
  { immediate: true, deep: true },
)

const nextValidator = computed(() => {
  if (filterByDays.value && (countDays.value < 30 || isNaN(countDays.value))) {
    return false
  }
  if (filterByWords.value && keyWords.value.length === 0) {
    return false
  }
  if (!filterByDays.value && !filterByWords.value && !filterByMoots.value && !filterBySpam.value) {
    return false
  }

  return true
})

const emit = defineEmits(['next'])

const onNext = () => {
  emit('next', {
    filterByDays: filterByDays.value,
    countDays: countDays.value,
    filterByWords: filterByWords.value,
    keyWords: keyWords.value,
    filterByMoots: filterByMoots.value,
    filterBySpam: filterBySpam.value,
    mode: mode.value,
  })
}
</script>

<template>
  <Dialog class="max-h-screen overflow-auto" modal v-bind:open="openConfig">
    <DialogContent class="max-h-screen overflow-auto">
      <DialogHeader class="space-y-6">
        <DialogTitle> Configurar filtros </DialogTitle>
        <DialogDescription class="space-y-6">
          <div>
            Escolha os filtros para remover os seguidores da sua conta. Após clicar em "Avançar"
            você poderá remover todos ou um por um.
          </div>
          <div class="flex flex-col gap-4">
            <Card class="flex flex-col gap-6 p-4">
              <div class="flex justify-between flex-wrap gap-2">
                <div class="flex gap-2 items-center text-start">
                  <IconCalendarX />
                  Por dias sem postar
                </div>
                <div class="flex items-center gap-2 ml-auto">
                  <Switch :checked="filterByDays" @update:checked="filterByDays = $event" />
                </div>
              </div>

              <NumberField
                class="gap-2"
                :min="30"
                :step="5"
                v-model="countDays"
                v-if="filterByDays"
              >
                <NumberFieldContent>
                  <NumberFieldDecrement />
                  <NumberFieldInput />
                  <NumberFieldIncrement />
                </NumberFieldContent>
              </NumberField>
            </Card>
            <Card class="flex flex-col gap-6 p-4">
              <div class="flex justify-between flex-wrap gap-2">
                <div class="flex gap-2 items-center text-start">
                  <IconFunnelSimpleX />
                  Por palavras-chaves na bio
                </div>
                <div class="flex items-center gap-2 ml-auto">
                  <Switch :checked="filterByWords" @update:checked="filterByWords = $event" />
                </div>
              </div>
              <div name="tags" v-if="filterByWords">
                <p class="text-sm text-gray-500 mb-2">
                  Separe por vírgula ou dê Enter ao final de cada palavra-chave.
                </p>
                <TagsInput v-model="keyWords">
                  <TagsInputItem v-for="item in keyWords" :key="item" :value="item">
                    <TagsInputItemText />
                    <TagsInputItemDelete />
                  </TagsInputItem>
                  <TagsInputInput placeholder="crypto..." />
                </TagsInput>
                <p class="text-sm text-gray-500 mt-2">
                  Você pode colocar palavras ou expressões como: crypto, trade, nsfw, adult content
                  creator, onlyfans.com, 🔞. Use com moderação.
                </p>
              </div>
            </Card>
            <Card class="flex flex-col gap-6 p-4">
              <div class="flex justify-between flex-wrap gap-2">
                <div class="flex gap-2 items-center text-start">
                  <IconUserCircleMinus />
                  Por quem não segue de volta
                </div>
                <div class="flex items-center gap-2 ml-auto">
                  <Switch :checked="filterByMoots" @update:checked="filterByMoots = $event" />
                </div>
              </div>
            </Card>
            <Card class="flex flex-col gap-6 p-4">
              <div class="flex justify-between flex-wrap gap-2">
                <div class="flex gap-2 items-center text-start">
                  <IconThumbsDown />
                  Por contas marcadas como SPAM
                </div>
                <div class="flex items-center gap-2 ml-auto">
                  <Switch :checked="filterBySpam" @update:checked="filterBySpam = $event" />
                </div>
              </div>
            </Card>
          </div>
          <Button class="w-full" :disabled="!nextValidator" @click="onNext"> Avançar </Button>
        </DialogDescription>
      </DialogHeader>
    </DialogContent>
  </Dialog>
</template>

<style></style>
