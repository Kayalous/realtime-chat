<template>
  <div>
    <amplify-authenticator>
      <!-- <amplify-sign-out></amplify-sign-out> -->
      <div
        class="flex flex-col flex-1 w-screen h-screen overflow-hidden  bg-dark-background"
      >
        <!-- Top bar -->
        <div
          class="flex items-center justify-between flex-none px-6 py-2 border-b border-gray-600 shadow-xl "
        >
          <div class="flex flex-col">
            <h3 class="mb-1 text-xl font-bold text-gray-100">
              <span class="text-gray-400">#</span> moshpit
            </h3>
          </div>
          <amplify-sign-out class="overflow-hidden rounded"></amplify-sign-out>
        </div>

        <!-- Chat messages -->
        <div class="flex-grow px-6 py-5 overflow-y-auto" id="messages">
          <transition-group name="list" tag="ul">
            <li
              v-for="message in messages"
              v-bind:key="message"
              class="flex items-start px-5 py-3 my-2 text-sm transition-all duration-200 rounded shadow  bg-dark-secondary filter hover:brightness-125"
            >
              <img
                :src="`https://avatars.dicebear.com/api/male/${message.username}.svg`"
                class="w-10 h-10 mr-3 cursor-pointer rounded-3xl"
              />
              <div class="flex-1 overflow-hidden">
                <div>
                  <span class="font-bold text-red-300">{{
                    message.username
                  }}</span>
                  <span class="ml-2 text-xs font-bold text-gray-400">{{
                    date(message.date)
                  }}</span>
                </div>
                <p class="leading-normal text-white">{{ message.body }}</p>
              </div>
            </li>
          </transition-group>
          <div ref="stopgap"></div>
        </div>
        <div
          class="flex items-center justify-center px-4 py-6 border-t  border-dark-secondary"
        >
          <div class="flex flex-grow overflow-hidden rounded-lg">
            <form class="w-full" v-on:submit.prevent="newMessage">
              <input
                ref="input"
                v-model="body"
                type="text"
                class="
                  w-full
                  px-4
                  text-white
                  border-2
                  focus:outline-none
                  focus:shadow-none
                  !border-dark-secondary
                  bg-dark-secondary
                  transition-all
                  duration-200
                  shadow
                  filter
                  hover:brightness-125
                  focus:hover:brightness-125
                  focus:brightness-110
                "
                placeholder="Message #moshpit"
              />
            </form>
          </div>

          <button
            @click="newMessage"
            class="
              w-10
              h-10
              p-2
              mx-5
              rounded-full
              bg-dark-secondary
              !outline-none
              transition-all
              duration-200
              shadow
              filter
              hover:brightness-125
            "
          >
            <svg
              xmlns="http://www.w3.org/2000/svg"
              class="text-gray-300 icon icon-tabler icon-tabler-brand-telegram"
              viewBox="0 0 24 24"
              stroke-width="1.5"
              stroke="currentColor"
              fill="none"
              stroke-linecap="round"
              stroke-linejoin="round"
            >
              <path stroke="none" d="M0 0h24v24H0z" fill="none" />
              <path d="M15 10l-4 4l6 6l4 -16l-18 7l4 2l2 6l3 -4" />
            </svg>
          </button>
        </div>
      </div>
    </amplify-authenticator>
  </div>
</template>

<script>
import { onAuthUIStateChange } from "@aws-amplify/ui-components";
import { DataStore } from "@aws-amplify/datastore";
import { Messages } from "./models";
import moment from "moment";
import { Auth, API, graphqlOperation } from "aws-amplify";
// import * as queries from "./graphql/queries";
// import * as mutations from "./graphql/mutations";
import * as subscriptions from "./graphql/subscriptions";

export default {
  name: "AuthStateApp",

  created() {
    this.unsubscribeAuth = onAuthUIStateChange((authState, authData) => {
      this.authState = authState;
      this.user = authData;
    });
  },
  data() {
    return {
      user: undefined,
      authState: undefined,
      unsubscribeAuth: undefined,
      messages: [],
      body: "",
    };
  },
  beforeUnmount() {
    this.unsubscribeAuth();
  },
  async mounted() {
    let currentUser = await Auth.currentAuthenticatedUser();
    let vm = this;
    API.graphql(
      graphqlOperation(subscriptions.onCreateMessages, {
        owner: currentUser.username,
      })
    ).subscribe({
      next: (action) => {
        vm.messages.push(action.value.data.onCreateMessages);
      },
    });

    this.messages = await DataStore.query(Messages);
    this.$nextTick(() => {
      this.$refs.stopgap.scrollIntoView({ behavior: "smooth" });
    });
  },

  methods: {
    async newMessage() {
      this.body = this.body.trim();
      if (this.body.length === 0) {
        this.$refs.input.focus();
        return;
      }
      await DataStore.save(
        new Messages({
          username: this.user.username,
          date: new Date().toLocaleString(),
          body: this.body,
        })
      );
      this.body = "";
    },

    date(date) {
      return moment(date).fromNow();
    },
  },

  watch: {
    messages: {
      handler: function () {
        this.$nextTick(() => {
          this.$refs.stopgap.scrollIntoView({ behavior: "smooth" });
        });
      },
      deep: true,
    },
  },
};
</script>


<style>
body {
  background: theme("colors.dark.background");
  min-height: 100vh !important;
  height: 100vh !important;
}
::-webkit-scrollbar {
  width: 8px;
  height: 3px;
}
::-webkit-scrollbar-button {
  /* display: none; */
}
::-webkit-scrollbar-track {
  background-color: theme("colors.dark.secondary");
}
::-webkit-scrollbar-track-piece {
  background-color: theme("colors.dark.background");
}
::-webkit-scrollbar-thumb {
  height: 50px;
  background-color: theme("colors.dark.secondary");
  border-radius: 3px;
}
::-webkit-scrollbar-corner {
  background-color: theme("colors.dark.background");
}
::-webkit-resizer {
  background-color: theme("colors.dark.secondary");
}

.list-enter-from {
  opacity: 0;
  transform: scale(0);
}
.list-enter-to {
  opacity: 1;
  transform: scale(1);
}
.list-enter-active {
  transition: all 0.5s cubic-bezier(0.34, 2, 0.6, 1);
}
</style>
