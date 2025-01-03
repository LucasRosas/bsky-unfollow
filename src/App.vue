<script setup>
import { ref } from 'vue'
import { z } from 'zod'
import { ScrollArea } from '@/components/ui/scroll-area'
import { Card, CardDescription, CardTitle } from '@/components/ui/card'
import { Button } from '@/components/ui/button'
import { Badge } from '@/components/ui/badge'
import { Separator } from '@/components/ui/separator'
import { Skeleton } from '@/components/ui/skeleton'
import { Progress } from '@/components/ui/progress'
import {
  Dialog,
  DialogContent,
  DialogDescription,
  DialogHeader,
  DialogTitle,
  DialogTrigger,
} from '@/components/ui/dialog'

import { AutoForm } from '@/components/ui/auto-form'
import { AtpAgent } from '@atproto/api'
import { Avatar, AvatarImage } from '@/components/ui/avatar'

import DialogConfig from '@/components/DialogConfig.vue'
const agent = new AtpAgent({
  service: 'https://bsky.social',
})

const openLogin = ref(true)
const openConfig = ref(false)
const login = ref({
  user: '',
  password: '',
})

const loginError = ref(null)

const followsCount = ref(0)
const realFollowersCount = ref(0)

const profile = ref({})
const tfaactive = ref(false)

const formSchema = z.object({
  user: z.string().describe('E-mail ou @'),
  password: z.string().min(1).describe('Senha'),
})
const TFASchema = z.object({
  TFA: z.string().describe('Código de login do Bluesky'),
})

import { useColorMode } from '@vueuse/core'
import IconCircleNotch from './components/icons/IconCircleNotch.vue'

const mode = useColorMode()
mode.value = 'dark'

const auth = ref(null)
const loadinLogin = ref(false)
const onSubmit = async (values) => {
  loadinLogin.value = true
  if (!tfaactive.value) login.value = values
  try {
    await agent.login({
      identifier: login.value.user,
      password: login.value.password,
      ...(tfaactive.value ? { authFactorToken: values.TFA } : {}),
    })
    let { data } = await agent.com.atproto.server.getSession()
    auth.value = data
  } catch (e) {
    if (e.message == 'A sign in code has been sent to your email address') {
      tfaactive.value = true
      loginError.value = null
      loadinLogin.value = false
      return
    }
    loginError.value = 'Usuário ou senha inválidos'
    loadinLogin.value = false
  }
  tfaactive.value = false
  let { data } = await agent.getProfile({
    actor: auth.value.did,
  })
  profile.value = data
  followsCount.value = data.followsCount
  openLogin.value = false
  openConfig.value = true
  loadinLogin.value = false
}

const wait = (ms) => new Promise((resolve) => setTimeout(resolve, ms))

const unfollow = async (did, viewer) => {
  let rkey = viewer.following.split('/').pop()
  await agent.com.atproto.repo.deleteRecord({
    repo: auth.value.did,
    collection: 'app.bsky.graph.follow',
    rkey: rkey,
  })
  let follower = followers.value.find((f) => f.did == did)
  follower.inactive = true
  followsCount.value -= 1
}

const unfollowAll = async () => {
  for await (const f of followersToUnfollow.value) {
    await unfollow(f.did, f.viewer)
    await wait(250)
  }
}

const follow = async (did) => {
  const { uri } = await agent.follow(did)
  let follower = followers.value.find((f) => f.did == did)
  follower.inactive = false
  follower.viewer.following = uri
  followsCount.value += 1
}

const followers = ref([])
const followersToUnfollow = ref([])
const cursor = ref(null)
const searchingFollowers = ref(undefined)
const progressFollowers = ref(0)
const evaluatingFollowers = ref(undefined)
const progressEvaluation = ref(0)

const getFollowers = async () => {
  followers.value = []
  followersToUnfollow.value = []
  searchingFollowers.value = true
  cursor.value = null
  do {
    const { data } = await agent.getFollows({
      actor: profile.value.handle,
      limit: 100,
      cursor: cursor.value,
    })
    const { follows } = data
    followers.value.push(...follows)
    cursor.value = data.cursor
    progressFollowers.value = (followers.value.length / followsCount.value) * 100
    await wait(250)
  } while (cursor.value != null)
  progressFollowers.value = 100
  realFollowersCount.value = followers.value.length
  searchingFollowers.value = false
}

const copiado = ref(false)
const copyCode = () => {
  navigator.clipboard
    .writeText(
      '00020101021126580014br.gov.bcb.pix013680aeb87a-9e21-4e66-a479-4344d8330b595204000053039865802BR5918LUCAS ARAUJO ROSAS6011SANTA LUZIA62070503***6304A630',
    )
    .then(() => {
      copiado.value = true
      setTimeout(() => {
        copiado.value = false
      }, 3000)
    })
}

const config = ref({
  countDays: 30,
  filterByDays: false,
  filterByMoots: false,
  filterByWords: false,
  filterBySpam: false,
  keyWords: [],
  mode: 'list',
})

const onNext = async (values) => {
  config.value = values
  openConfig.value = false
  await getFollowers()
  evaluateFollowers()
}

const feedIsEmpty = async (handle) => {
  if (handle == 'handle.invalid') {
    return true
  }
  await wait(110)
  const { data } = await agent.getAuthorFeed({
    actor: handle,
    limit: 2,
  })

  const { feed } = data

  if (feed.length == 0) {
    return true
  }
  if (feed[0].reason?.indexedAt) {
    return (
      new Date(feed[0].reason.indexedAt).valueOf() <
      new Date().valueOf() - 1000 * 60 * 60 * 24 * config.value.countDays
    )
  }

  return (
    new Date(feed[0].post.indexedAt).valueOf() <
    new Date().valueOf() - 1000 * 60 * 60 * 24 * config.value.countDays
  )
}

const evaluateFollowers = async () => {
  evaluatingFollowers.value = true
  progressEvaluation.value = 0
  let count = 0
  for await (const f of followers.value) {
    count++
    progressEvaluation.value = (count / realFollowersCount.value) * 100
    if (f.handle == 'handle.invalid') {
      f.unfollow = true
      continue
    }
    if (config.value.filterByDays && (await feedIsEmpty(f.handle))) {
      f.unfollow = true
      continue
    }
    if (config.value.filterByMoots && !f.viewer.followedBy) {
      f.unfollow = true
      continue
    }

    if (config.value.filterBySpam && f.labels.some((i) => i.val == 'spam')) {
      f.unfollow = true
      continue
    }
    if (
      config.value.filterByWords &&
      config.value.keyWords
        .map((i) => i.toLowerCase())
        .some((k) => {
          if (f.description == undefined) return false
          return f.description.toLowerCase().includes(k)
        })
    ) {
      f.unfollow = true
      continue
    }
  }
  followersToUnfollow.value = followers.value.filter((f) => f.unfollow)
  evaluatingFollowers.value = false
}
</script>

<template>
  <Dialog defaultOpen modal v-bind:open="openLogin">
    <DialogContent>
      <DialogHeader class="space-y-6">
        <DialogTitle> Sai fora usuário inativo! </DialogTitle>
        <DialogDescription class="space-y-6">
          <div>
            ✨✨ Para que seja possível remover os seguidores inativos da sua conta é necessário que
            você esteja logado. Não se preocupe, seus dados estão seguros! Não salvamos nada em
            nossos servidores. Em breve vamos melhorar a forma de te conectar com o bsky ❤️.
          </div>

          <AutoForm
            v-if="!tfaactive"
            class="space-y-6"
            :schema="formSchema"
            :field-config="{
              user: {
                inputProps: {
                  type: 'email',
                  placeholder: 'meu@email.com ou meuarroba.bsky.social',
                },
              },
              password: {
                label: 'Senha',
                inputProps: {
                  type: 'password',
                  placeholder: '••••••••',
                },
              },
            }"
            @submit="onSubmit"
          >
            <div class="text-red-500" v-if="loginError">
              {{ loginError }}
            </div>
            <div class="flex align-end flex-col space-y-4 gap-4">
              <Button type="submit" :disabled="loadinLogin">
                <IconCircleNotch v-if="loadinLogin" class="animate-spin" />
                Avançar
              </Button>
            </div>
          </AutoForm>
          <AutoForm
            v-if="tfaactive"
            class="space-y-6"
            :schema="TFASchema"
            :field-config="{
              TFA: {
                label: 'Código de autenticação',
                hidden: !tfaactive,
                inputProps: {
                  type: 'text',
                  placeholder: '••••••••',
                },
              },
            }"
            @submit="onSubmit"
          >
            <div class="text-red-500" v-if="loginError">
              {{ loginError }}
            </div>
            <div class="flex align-end flex-col space-y-4 gap-4">
              <Button type="submit"> Avançar </Button>
            </div>
          </AutoForm>
        </DialogDescription>
      </DialogHeader>
    </DialogContent>
  </Dialog>

  <DialogConfig :openConfig="openConfig" :config="config" @next="onNext" />

  <main v-if="!openLogin">
    <div class="p-2 space-y-4">
      <Card class="p-4 space-y-3">
        <CardTitle class="flex items-center space-x-1 gap-3">
          <Avatar>
            <AvatarImage :src="profile.avatar" alt="@radix-vue" />
          </Avatar>
          <div>
            {{ profile.displayName }}
            <div class="text-sm font-thin tracking-wide">@{{ profile.handle }}</div>
          </div>
        </CardTitle>
        <CardDescription class="space-y-3">
          <div class="flex space-x-4">
            <div>
              <strong>{{ profile.followersCount }}</strong> seguidores
            </div>
            <div>
              <strong>{{ followsCount }}</strong> seguindo
            </div>

            <div>
              <strong>{{ profile.postsCount }}</strong> posts
            </div>
          </div>
          <Card
            class="border-red-800 px-2 py-2"
            v-if="realFollowersCount && followsCount - realFollowersCount > 0"
          >
            Das <strong>{{ followsCount }}</strong> contas que você segue
            <strong>{{ followsCount - realFollowersCount }}</strong> são contas excluídas ou
            bloqueadas. Sobram <strong>{{ realFollowersCount }}</strong> contas não deletadas. O
            Bluesky ainda não permite parar de seguir contas excluídas. 🤷‍♂️ (bug reportado
            <a target="_blank" href="https://github.com/bluesky-social/atproto/issues/2525">aqui</a>
            )
          </Card>
          <div
            class="px-2 py-2 bg-slate-900 rounded-lg"
            v-html="profile.description.replaceAll('\n', '<br />')"
          />
        </CardDescription>
      </Card>
      <div v-if="searchingFollowers" class="p-4 space-y-4">
        Buscando usuários que você segue... ({{ progressFollowers.toFixed(1) }}%)
        <Progress class="h-2" :model-value="progressFollowers" />
      </div>
      <div v-if="evaluatingFollowers" class="p-4 space-y-4">
        Analisando usuários que você segue... ({{ progressEvaluation.toFixed(1) }}%)
        <Progress class="h-2" :model-value="progressEvaluation" />
      </div>
      <div class="flex justify-center">
        <Button size="sm" @click="openConfig = true">Refazer filtro</Button>
      </div>
      <div v-if="!evaluatingFollowers && !searchingFollowers">
        Listando usuários que você segue e que
        <ul class="list-disc list-inside p-4">
          <li>Possuem @ inválido</li>
          <li v-if="config.filterByDays">Pão postam há mais de {{ config.countDays }} dias</li>
          <li v-if="config.filterByMoots">Não te seguem de volta</li>
          <li v-if="config.filterBySpam">Foram marcados como SPAM</li>
          <li v-if="config.filterByWords">
            Têm as palavras-chave na bio: {{ config.keyWords.join(', ') }}
          </li>
        </ul>

        Foram encontrados <strong>{{ followersToUnfollow.length }}</strong> usuários com esses
        filtros.
      </div>

      <ScrollArea
        class="h-[50vh] rounded-md border border-red-500 px-4 space-y-3"
        v-if="followersToUnfollow.length > 0"
      >
        <template v-for="follower in followersToUnfollow" :key="follower.did">
          <div class="flex justify-between items-center flex-wrap p-2">
            <CardTitle class="flex space-x-1 gap-3" :class="{ 'opacity-20': follower.inactive }">
              <Avatar>
                <AvatarImage :src="follower.avatar" alt="@radix-vue" />
              </Avatar>
              <div>
                {{ follower.displayName }}
                <div class="text-sm font-thin tracking-wide">@{{ follower.handle }}</div>
                <Badge variant="secondary" v-if="follower.viewer.followedBy">Segue você</Badge>
                <Badge variant="destructive" v-if="follower.labels.some((i) => i.val == 'spam')"
                  >SPAM</Badge
                >
                <Badge variant="destructive" v-if="follower.handle == 'handle.invalid'"
                  >Usuário Inválido</Badge
                >
              </div>
            </CardTitle>
            <div
              class="flex items-center justify-end ml-auto space-x-4 text-right"
              v-if="follower.unfollow"
            >
              <Button
                variant="destructive"
                size="sm"
                class="justify-self-end ml-auto"
                @click="unfollow(follower.did, follower.viewer)"
                v-if="!follower.inactive"
                >Parar de seguir</Button
              >
              <Button
                size="sm"
                class="justify-self-end ml-auto"
                @click="follow(follower.did, follower.viewer)"
                v-else
                >Seguir</Button
              >
            </div>
          </div>
          <Separator />
        </template>
        <div class="flex items-center space-x-4 p-2" v-if="cursor != null">
          <Skeleton class="h-12 w-12 rounded-full" />
          <div class="space-y-2">
            <Skeleton class="h-4 w-[250px]" />
            <Skeleton class="h-4 w-[200px]" />
          </div>
        </div>
      </ScrollArea>
      <div class="flex justify-end" v-if="followersToUnfollow.length > 0">
        <Button variant="destructive" size="sm" @click="unfollowAll">Parar de seguir todos</Button>
      </div>
    </div>
    <div class="p-4 text-center text-xs text-gray-500">
      Feito com ❤️ por
      <a href="https://bsky.app/profile/lucasaros.bsky.social"> @lucasaros.bsky.social </a>

      <div class="p-4 text-center text-xs text-gray-500">
        <Dialog>
          <DialogTrigger as-child>
            <Button size="sm">☕ Apoie-me com um pix</Button>
          </DialogTrigger>
          <DialogContent>
            <DialogHeader class="space-y-6">
              <DialogTitle> ☕ Apoie-me com um pix </DialogTitle>
              <DialogDescription class="space-y-6">
                <div>
                  Se você quiser me ajudar a manter esse script funcionando e atualizado, você pode
                  fazer um pix pra mim com o valor que quiser. Obrigado!
                </div>
                <div class="flex justify-center items-center">
                  <Button size="sm" variant="secondary" @click="copyCode">{{
                    copiado ? 'Copiado!' : 'Copiar código'
                  }}</Button>
                </div>
                <img src="/image.png" alt="" srcset="" />
              </DialogDescription>
            </DialogHeader>
          </DialogContent>
        </Dialog>
      </div>
    </div>
  </main>
</template>

<style>
#app {
  padding: 1rem;
}
</style>
