<template>
  <div>
    <div class="title">
      <h2>Amplifyで作るチャットアプリ</h2>
    </div>
    <div class="main-contents">
      <div class="message_base">
        <div v-for="message in messages" :key="message.id">
          <div
            v-bind:class="[
              message.username === userName ? 'message' : 'message_opponent',
            ]"
          >
            {{ message.content }}
          </div>
          <div
            v-bind:class="[
              message.username === userName ? 'username' : 'username_opponent',
            ]"
          >
            {{ message.username }}
          </div>
        </div>
      </div>
      <el-input
        placeholder="メッセージを入力(Enterで送信)"
        v-model="content"
        @keydown.enter.native="sendMessage"
      ></el-input>
      <div class="error">{{ this.error }}</div>
    </div>
  </div>
</template>

<script>
import API, { graphqlOperation } from "@aws-amplify/api";
import { Auth } from "aws-amplify";
import { createMessage } from "./graphql/mutations";
import { listMessages } from "./graphql/queries";
import { onCreateMessage } from "./graphql/subscriptions";
window.LOG_LEVEL = "VERBOSE";
export default {
  name: "chat",
  data() {
    return {
      messages: [],
      content: "",
      userName: "",
      subscription: {},
      error: "",
    };
  },
  methods: {
    // TODO(1) GraphQLエンドポイントにMutationを発行し、メッセージを登録する
    sendMessage() {
      if (event.keyCode !== 13) return; // Enterキーの場合のみ送信処理を行う
      const message = {
        id: new Date().getTime() + this.userName,
        username: this.userName,
        content: this.content,
      };
      // Mutationの実行
      API.graphql(graphqlOperation(createMessage, { input: message })).catch(
        (error) => (this.error = JSON.stringify(error))
      );
      this.content = "";
    },
    // TODO(2) GraphQLエンドポイントにQueryを発行し、メッセージ一覧を取得する
    fetch() {
      API.graphql(graphqlOperation(listMessages, { limit: 100 }))
        .then(
          (messages) =>
            (this.messages = messages.data.listMessages.items.sort((a, b) =>
              a.id > b.id ? 1 : -1
            ))
        )
        .catch((error) => (this.error = JSON.stringify(error)));
    },
    // TODO(3-1) GraphQLエンドポイントにSubscriptionを発行し、mutationを監視する
    subscribe() {
      this.subscription = API.graphql(
        graphqlOperation(onCreateMessage)
      ).subscribe({
        next: (eventData) => {
          const message = eventData.value.data.onCreateMessage;
          this.messages.push(message);
        },
      });
    },
    scrollBottom() {
      const container = this.$el.querySelector(".message_base");
      container.scrollTop = container.scrollHeight;
    },
  },
  async created() {
    this.userName = (await Auth.currentAuthenticatedUser()).username;
    this.fetch();
    this.subscribe();
  },
  // TODO(3-2) チャット画面から離れる際に、UnSubscribeする
  beforeDestroy() {
    this.subscription.unsubscribe();
  },
  updated: function () {
    this.scrollBottom();
  },
};
</script>

<style src="./assets/chat.css" />