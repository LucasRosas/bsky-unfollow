<script setup>
import { ref } from 'vue'
import { z } from 'zod'
import { ScrollArea } from '@/components/ui/scroll-area'
import { Card, CardDescription, CardTitle } from '@/components/ui/card'
import { Button } from '@/components/ui/button'
import { Separator } from '@/components/ui/separator'
import { Skeleton } from '@/components/ui/skeleton'
import {
  Dialog,
  DialogContent,
  DialogDescription,
  DialogHeader,
  DialogTitle,
} from '@/components/ui/dialog'

import { AutoForm } from '@/components/ui/auto-form'
import { BskyAgent } from '@atproto/api'
import { Avatar, AvatarImage } from '@/components/ui/avatar'
const agent = new BskyAgent({
  service: 'https://bsky.social',
})

const openLogin = ref(true)
const login = ref({
  email: '',
  password: '',
  arroba: '',
})

const followsCount = ref(0)

const profile = ref({})

const formSchema = z.object({
  email: z.string().email().describe('E-mail'),
  password: z.string().min(1).describe('Senha'),
  arroba: z.string().min(1).describe('Arroba'),
})

import { useColorMode } from '@vueuse/core'

const mode = useColorMode()
mode.value = 'dark'

const interactions = ref(0)
const count = ref(0)

const onSubmit = async (values) => {
  login.value = values
  await agent.login({
    identifier: login.value.email,
    password: login.value.password,
  })
  const { data } = await agent.getProfile({
    actor: login.value.arroba,
  })
  console.log(data)
  profile.value = data
  followsCount.value = data.followsCount

  interactions.value = Math.ceil(profile.value.followersCount / 100)
  count.value = 0
  openLogin.value = false
  initRemotion()
}

const wait = (ms) => new Promise((resolve) => setTimeout(resolve, ms))
const cursor = ref(null)

const unfollow = async (did, handle) => {
  const { uri } = await agent.follow(did)
  await agent.deleteFollow(uri)
  console.log('Deixei de seguir ', handle)
  let follower = followers.value.find((f) => f.did == did)
  follower.inactive = true
  followsCount.value -= 1
}

const followers = ref([])

const initRemotion = async () => {
  do {
    const { data } = await agent.getFollows({
      actor: profile.value.handle,
      limit: 100,
      cursor: cursor.value,
    })

    console.log('Seguidores obtidos com sucesso')

    const { follows } = data
    followers.value.push(...follows)
    cursor.value = data.cursor
    for await (const follow of follows) {
      let { did, handle } = follow
      console.log(`\n Buscando feed de ${handle}`)
      const { data } = await agent.getAuthorFeed({
        actor: handle,
        limit: 1,
      })

      const { feed } = data

      if (feed.length == 0) {
        console.log('feed is empty')
        await unfollow(did, handle)
      } else if (
        new Date(feed[0].post.indexedAt).valueOf() <
        new Date().valueOf() - 1000 * 60 * 60 * 24 * 30
      ) {
        await unfollow(did, handle)

        console.log('feed older than 30 days')
      } else {
        console.log('feed is up to date')
        let follower = followers.value.find((f) => f.did == did)
        follower.inactive = false
      }
      await wait(250)
    }
    await wait(250)
    count.value++
  } while (cursor.value != null)
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
            class="space-y-6"
            :schema="formSchema"
            :field-config="{
              email: {
                inputProps: {
                  type: 'email',
                  placeholder: 'meu@email.com',
                },
              },
              password: {
                label: 'Senha',
                inputProps: {
                  type: 'password',
                  placeholder: '••••••••',
                },
              },
              arroba: {
                inputProps: {
                  type: 'string',
                  placeholder: '@meuarroba.bsky.social',
                },
              },
            }"
            @submit="onSubmit"
          >
            <div class="flex align-end flex-col space-y-4 gap-4">
              Ao clicar em entrar o script irá buscar seus seguidores e verificar se estão ativos.
              Os inativos serão removidos. Você concorda com isso?
              <Button type="submit"> Entrar </Button>
            </div>
          </AutoForm>
        </DialogDescription>
      </DialogHeader>
    </DialogContent>
  </Dialog>

  <main v-if="!openLogin">
    <div class="p-4 space-y-4">
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
          <div
            class="px-2 py-2 bg-slate-900 rounded-lg"
            v-html="profile.description.replaceAll('\n', '<br />')"
          />
        </CardDescription>
      </Card>

      <ScrollArea class="h-[50vh] rounded-md border px-4 space-y-3">
        <template v-for="follower in followers" :key="follower.did">
          <div
            class="flex justify-between items-center p-2"
            :class="{ 'opacity-20': follower.inactive }"
          >
            <CardTitle class="flex items-center space-x-1 gap-3">
              <Avatar>
                <AvatarImage :src="follower.avatar" alt="@radix-vue" />
              </Avatar>
              <div>
                {{ follower.displayName }}
                <div class="text-sm font-thin tracking-wide">@{{ follower.handle }}</div>
              </div>
            </CardTitle>
            {{
              follower.inactive == undefined
                ? 'Buscando dados...'
                : follower.inactive
                  ? 'Usuário inativo: parando de seguir'
                  : 'Usuário ativo'
            }}
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
    </div>
    <div class="p-4 text-center text-xs text-gray-500">
      Feito com ❤️ por
      <a href="https://bsky.app/profile/lucasaros.bsky.social"> @lucasaros.bsky.social </a>
    </div>
  </main>
</template>

<style>
#app {
  padding: 1rem;
}
</style>
