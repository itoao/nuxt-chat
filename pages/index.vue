<template>
  <div style="max-width: 800px;">
    <amplify-authenticator>
      <v-text-field
      v-model="form.comment"
      label="コメント"
      placeholder="ここにコメントを書きましょう"
      outlined
      class="mx-auto"
      append-icon="mdi-check-bold"
      style="max-width: 100%; box-sizing: border-box;"
      @keydown="onEnter"
      @click:append="createChat"
      ></v-text-field>
    </amplify-authenticator>

    <v-card v-for="(item, index) in items" :key="index" tile>
      <v-list-item two-line>
        <v-list-item-content>
          <v-list-item-title>{{ item.comment }}</v-list-item-title>
          <v-list-item-subtitle>by: {{ item.owner }}</v-list-item-subtitle>
        </v-list-item-content>
      </v-list-item>
    </v-card>

    <v-card>
      <amplify-sign-out v-if="logoutBtn"></amplify-sign-out>
    </v-card>
  </div>
</template>
<script>
import { onAuthUIStateChange } from '@aws-amplify/ui-components'
import { API, graphqlOperation } from 'aws-amplify' // Amplifyライブラリを読み込み
import { createChat } from '~/src/graphql/mutations' // GraphQL Mutation（データをエンドポイントに送信する構文?）
import { listChats } from '~/src/graphql/queries' // GraphQL Query（データを読み込む構文？）
import { onCreateChat } from '~/src/graphql/subscriptions' // GraphQL Subscription

export default {
  data() {
    return {
      form: {
        comment: '',
      },
      items: [],
      logoutBtn: false,
      username: '',
    }
  },
  created() {
    this.getChatList()
    // this.subscribe() // 追加
    onAuthUIStateChange((authState, authData) => {
      if(authState === 'signedin') {
        this.username = authData.username
        this.getChatList()
        this.subscribe()
        this.logoutBtn = true
      } else {
        this.item = []
        this.logoutBtn = false
      }
    })
  },
  methods: {
    subscribe() {
      API.graphql(
        graphqlOperation(onCreateChat, { owner: this.username })
      ).subscribe({
        next: (e) => {
          const chat = e.value.data.onCreateChat
          if(this.items.some((item) => item.comment === chat.comment)) return
          this.items = [...this.items, chat]
        }
      })
    },
    async createChat() {
      // コメントを送信する
      const comment = this.form.comment // コメント入力値を取得
      if (!comment) return // 空のときは処理しない
      const chat = { comment } // 送信用のJSONを作成
      // 送信処理
      await API.graphql({
        query: createChat, // GraphQL Mutation
        variables: { input: chat }, // 送信データ
      })
      this.form.comment = '' // 送信後にテキストフィールドを空に。
    },
    onEnter(event) {
      // ここはおまけ。（Enterを押したときもコメントを送信したかったので記述）
      if (event.keyCode !== 13) return
      this.createChat()
    },
    async getChatList() {
      // コメント一覧を取得
      const chatList = await API.graphql({
        query: listChats, // GraphQL Query
      })
      this.items = chatList.data.listChats.items // 読み込みしたデータを一覧に表示
    },
  },
}
</script>
