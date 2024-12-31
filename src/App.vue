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
  DialogTrigger,
} from '@/components/ui/dialog'

import { AutoForm } from '@/components/ui/auto-form'
import { AtpAgent } from '@atproto/api'
import { Avatar, AvatarImage } from '@/components/ui/avatar'
const agent = new AtpAgent({
  service: 'https://bsky.social',
})

const openLogin = ref(true)
const login = ref({
  user: '',
  password: '',
})

const loginError = ref(null)

const followsCount = ref(0)

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

const mode = useColorMode()
mode.value = 'dark'

const interactions = ref(0)
const count = ref(0)
const auth = ref(null)
const onSubmit = async (values) => {
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
      return
    }
    loginError.value = 'Usuário ou senha inválidos'
  }
  tfaactive.value = false
  let { data } = await agent.getProfile({
    actor: auth.value.did,
  })
  profile.value = data
  followsCount.value = data.followsCount

  interactions.value = Math.ceil(profile.value.followersCount / 100)
  count.value = 0
  openLogin.value = false
  initRemotion()
}

const wait = (ms) => new Promise((resolve) => setTimeout(resolve, ms))
const cursor = ref(null)

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

const followers = ref([])

const initRemotion = async () => {
  do {
    const { data } = await agent.getFollows({
      actor: profile.value.handle,
      limit: 100,
      cursor: cursor.value,
    })

    const { follows } = data
    followers.value.push(...follows)
    cursor.value = data.cursor
    for await (const follow of follows) {
      let { did, handle, viewer } = follow
      const { data } = await agent.getAuthorFeed({
        actor: handle,
        limit: 2,
      })

      const { feed } = data

      if (feed.length == 0) {
        await unfollow(did, viewer)
      } else if (
        new Date(feed[0].post.indexedAt).valueOf() <
        new Date().valueOf() - 1000 * 60 * 60 * 24 * 45
      ) {
        await unfollow(did, viewer)
      } else {
        let follower = followers.value.find((f) => f.did == did)
        follower.inactive = false
      }
      await wait(250)
    }
    await wait(250)
    count.value++
  } while (cursor.value != null)
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
          <div>
            Os usuários inativos são aqueles que nunca postaram nada ou não postam a mais de 45
            dias.
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
              Ao clicar em entrar o script irá buscar seus seguidores e verificar se estão ativos.
              Os inativos serão removidos. Você concorda com isso?
              <Button type="submit"> Entrar </Button>
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
