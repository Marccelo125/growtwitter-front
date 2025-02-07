<script setup lang="ts">
import type { TweetType } from '@/types';
import default_avatar from '@/assets/default-avatar.png';
import { tempoDesdeCriacao } from '@/utils/PastTime';
import { deleteTweet, postComment, postLike, postRetweet } from '@/services/api';
import { onMounted, ref } from 'vue';
import IconComment from './icons/IconComment.vue';
import ModalSeeProfile from '@/components/ModalSeeProfile.vue';
import { getUserData, getUserId } from '@/services/authentication';

interface TweetTypeProps {
  data: TweetType;
  yourProfile?: boolean;
  isaReTweet?: boolean;
}

const props = defineProps<TweetTypeProps>();

const cardToList = defineEmits(['toListCard', 'rtListCard']);

const liked = ref(false);
const artificialLike = ref(0);
const retweetDrop = ref(false);
const tweetDrop = ref(false);
const retweetModal = ref(false);
const comment = ref('');
const retweetLoading = ref(false);
const commentInput = ref<string>('');
const showDiv = ref(false);
const userID = getUserId();
const me = ref(JSON.parse(localStorage.getItem('userData') || '{}'));
const localComments = ref([...props.data.comments]);
const localCommentsCount = ref(props.data.comments_count);

function like() {
  if (liked.value === false) {
    artificialLike.value++;
    liked.value = true;
  } else {
    artificialLike.value--;
    liked.value = false;
  }
}

function toList(id: number, isTweet: boolean) {
  cardToList('toListCard', {
    id,
    isTweet
  });
}

function rtToList(retweet: any, post: any, user: any) {
  const formattedRt = Object.assign(retweet, { post: post }, { user: user });
  cardToList('rtListCard', formattedRt);
}

async function handleSubmit(id: number) {
  if (commentInput.value.length > 0) {
    await postComment(id, commentInput.value);
    localComments.value.push({
      id: Math.floor(Math.random() * 10000),
      user: {
        id: id,
        avatar_url: me.value.avatar_url,
        name: me.value.name,
        username: me.value.username,
        surname: '',
        email: '',
        password: '',
        following_count: 0,
        followers_count: 0,
        posts_count: 0,
        retweets_count: 0,
        followers: [],
        followings: [],
        posts: [],
        retweets: [],
        cover_url: me.value.cover_url
      },

      content: commentInput.value,
      created_at: new Date().toISOString()
    });
    localCommentsCount.value++;
    commentInput.value = '';
  }
}

const toogleDiv = () => {
  showDiv.value = !showDiv.value;
};

const toggleReTweetDrop = () => {
  retweetDrop.value = !retweetDrop.value;
};

const toggleTweetDrop = () => {
  tweetDrop.value = !tweetDrop.value;
};

const handleRetweet = async (id: number) => {
  toggleReTweetDrop();
  const response = await postRetweet(id);
  rtToList(response.data.data, props.data, getUserData());
};

const handleRetweetWithComment = async (id: number, content: string) => {
  retweetLoading.value = true;
  toggleReTweetDrop();
  const response = await postRetweet(id, content);
  rtToList(response.data.data, props.data, getUserData());
  if (response) {
    retweetModal.value = false;
  }
};

const handleDeleteTweet = async (postID: number) => {
  toggleTweetDrop();
  toList(postID, true);
  await deleteTweet(postID);
  return;
};

const toggleModalRetweet = () => {
  retweetModal.value = true;
  comment.value = '';
};

async function handlePostLike(id: number) {
  like();
  let resp = await postLike(id);
  if (!resp) {
    like();
  }
}

onMounted(() => {
  const user = localStorage.getItem('userData');
  if (user) {
    liked.value = props.data.likes.some((like: any) => like.userId == JSON.parse(user).id);
  }
});
</script>

<template>
  <div class="card-principal rounded-0">
    <v-card-actions>
      <div class="profileModal d-block align-self-start">
        <RouterLink :to="`/profile/${data.user.id}`">
          <v-avatar :image="data.user.avatar_url ?? default_avatar" size="50"></v-avatar>
        </RouterLink>
        <ModalSeeProfile class="profileModalChild" style="z-index: 9999" :data="props.data.user" />
      </div>
      <div class="d-flex flex-column justify-space-between w-100 ml-2">
        <div class="d-flex align-center justify-space-between w-100">
          <div>
            <RouterLink :to="`/profile/${data.user.id}`" aria-label="User profile">
              <strong class="mouseHover">{{ data.user.name }}</strong> <span style="color: #657786">@{{ data.user.username }}</span>
            </RouterLink>
            <span> ·</span> <span>{{ tempoDesdeCriacao(data.created_at) }}</span>
            <p class="tweet-content">{{ data.content }}</p>
          </div>
          <div v-if="!isaReTweet" style="display: flex; align-items: end; flex-direction: column; position: relative">
            <v-btn @click="toggleTweetDrop" icon="mdi-dots-vertical"> </v-btn>
            <div v-if="tweetDrop" class="delTweet">
              <v-btn class="text-none" v-if="userID === data.user.id" @click="handleDeleteTweet(data.id)">Apagar</v-btn>
              <v-btn class="text-none" v-else>Denunciar</v-btn>
            </div>
          </div>
        </div>
        <div class="d-flex ga-1">
          <article class="align-center text-center">
            <v-btn icon class="btn-comment" @click="toogleDiv()">
              <IconComment class="icon-comment" />
              <span>
                {{ localCommentsCount }}
              </span>
            </v-btn>
          </article>

          <article class="d-flex">
            <v-btn icon v-if="!liked" class="btn-like d-flex align-center ga-0" @click="handlePostLike(data.id)">
              <v-icon icon="mdi-cards-heart-outline" />
              <span>
                {{ data.likes_count + artificialLike }}
              </span>
            </v-btn>

            <v-btn icon v-if="liked" class="btn-like d-flex align-center ga-0" @click="handlePostLike(data.id)">
              <v-icon icon="mdi-cards-heart" color="#f91880" />
              <span>
                {{ data.likes_count + artificialLike }}
              </span>
            </v-btn>

            <div v-if="!isaReTweet" style="display: flex; align-items: end; flex-direction: column; position: relative">
              <v-btn @click="toggleReTweetDrop" icon="mdi-repeat-variant" class="btn-reTweet"> </v-btn>
              <div v-if="retweetDrop" class="dropdown">
                <v-btn @click="handleRetweet(data.id)">Retweetar</v-btn>
                <v-btn @click="toggleModalRetweet">Retweetar com comentário</v-btn>
              </div>
            </div>
          </article>
        </div>
        <div v-if="showDiv" class="mt-2">
          <div v-for="comment in localComments" :key="comment.id" class="d-flex flex-column pb-4">
            <div class="d-flex align-center w-100 mx-2">
              <div class="profileModal mx-2">
                <RouterLink :to="`/profile/${comment.user.id}`">
                  <v-avatar :image="comment.user.avatar_url ?? default_avatar" size="45"></v-avatar>
                </RouterLink>
                <ModalSeeProfile class="profileModalChild" style="z-index: 9999" :data="comment.user" />
              </div>

              <div class="mx-2">
                <div>
                  <RouterLink :to="`/profile/${comment.user.id}`" aria-label="User profile">
                    <span class="mouseHover font-weight-bold">{{ comment.user.name }}</span> <span style="color: #657786">@{{ comment.user.username }}</span>
                  </RouterLink>

                  <span style="color: #657786"> ·</span> <span style="color: #657786">{{ tempoDesdeCriacao(comment.created_at) }}</span>
                </div>
                <div class="d-flex align-center w-100 mt-0 text-h7">
                  {{ comment.content }}
                </div>
              </div>
            </div>
          </div>
          <form @submit.prevent="handleSubmit(data.id)">
            <div class="text-box">
              <div class="box-container">
                <textarea placeholder="Comentar" v-model="commentInput" aria-label="Add a comment" aria-required="true"></textarea>

                <div class="formatting">
                  <button type="submit" class="send" title="Send" aria-label="Send comment">
                    <svg fill="none" viewBox="0 0 24 24" height="18" width="18" xmlns="http://www.w3.org/2000/svg">
                      <path stroke-linejoin="round" stroke-linecap="round" stroke-width="2.5" stroke="#ffffff" d="M12 5L12 20"></path>
                      <path
                        stroke-linejoin="round"
                        stroke-linecap="round"
                        stroke-width="2.5"
                        stroke="#ffffff"
                        d="M7 9L11.2929 4.70711C11.6262 4.37377 11.7929 4.20711 12 4.20711C12.2071 4.20711 12.3738 4.37377 12.7071 4.70711L17 9"
                      ></path>
                    </svg>
                  </button>
                </div>
              </div>
            </div>
          </form>
        </div>
      </div>
    </v-card-actions>
  </div>

  <v-dialog v-model="retweetModal" max-width="500px" aria-labelledby="retweet-dialog-title" aria-modal="true">
    <v-card>
      <v-card-title>
        <span class="text-h5" id="retweet-dialog-title">Retweetar com comentário</span>
      </v-card-title>
      <v-card-text>
        <v-textarea v-model="comment" id="comment-textarea" label="Adicionar um comentário..." auto-grow rows="3" aria-label="Add a comment" aria-labelledby="comment-label" aria-required="true"></v-textarea>
        <v-card class="mt-4" outlined>
          <v-card-text>
            <div class="d-flex align-center">
              <v-avatar :image="data.user.avatar_url ?? default_avatar" size="45" aria-label="User avatar" role="img"></v-avatar>
              <div class="ml-3">
                <div class="mouseHover font-weight-bold">{{ data.user.name }}</div>
                <div class="text--secondary">@{{ data.user.username }}</div>
              </div>
            </div>
            <div class="mt-3 ml-13">{{ data.content }}</div>
          </v-card-text>
        </v-card>
      </v-card-text>
      <v-card-actions>
        <v-spacer></v-spacer>
        <v-btn color="blue darken-1" @click="retweetModal = false" aria-label="Cancel">Cancelar</v-btn>
        <v-btn :loading="retweetLoading" color="blue darken-1" @click="handleRetweetWithComment(data.id, comment)" aria-label="Retweet">Retweet</v-btn>
      </v-card-actions>
    </v-card>
  </v-dialog>
</template>

<style scoped>
.profileModalChild {
  display: none;
}

.profileModal:hover .profileModalChild {
  display: flex;
}

.icon-comment {
  fill: #808080 !important;
}

.btn-comment:hover {
  color: #2c8cd4 !important;
  fill: #2c8cd4;
}

.btn-comment:hover .icon-comment {
  color: #2c8cd4 !important;
  fill: #2c8cd4 !important;
}

.btn-like {
  text-transform: none !important;
  user-select: none;
  cursor: pointer;
}

.btn-reTweet {
  text-transform: none !important;
  user-select: none;
  cursor: pointer;
  color: #808080;
}

.btn-reTweet:hover {
  color: #2c8cd4 !important;
  fill: #2c8cd4 !important;
}

.btn-like:hover {
  color: #f91880 !important;
}

.btn-like:hover .mdi-cards-heart-outline {
  color: #f91880 !important;
}

.mdi-cards-heart-outline {
  color: #808080;
}

.mouseHover {
  transition: all 0.2s ease;
}

.mouseHover:hover {
  text-decoration: underline;
}

.card-principal {
  border-top: 1px solid #ebe8e8;
  transition: background-color 0.3s ease;
}

.dropdown {
  display: flex;
  flex-direction: column;
  left: -100px;
  top: 50px;
  position: absolute;
  background-color: white;
  border: 1px solid #ebe8e8;
  padding: 10px;
  border-radius: 5px;
  width: max-content;
  z-index: 10;
}

.delTweet {
  background-color: #026eda;
  display: flex;
  flex-direction: column;
  top: 50px;
  right: 0px;
  position: absolute;
  background-color: white;
  border: 1px solid #ebe8e8;
  padding: 10px;
  border-radius: 5px;
  width: max-content;
  z-index: 10;
}

.tweet-body {
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  width: 100%;
}

.tweet-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  width: 100%;
}

.tweet-header strong {
  font-weight: bold;
}

.tweet-header span {
  color: #657786;
  font-size: 14px;
}

.tweet-content {
  overflow-wrap: break-word;
  word-break: break-word;
}

.text-box {
  width: 100%;
  height: fit-content;
  padding: 8px;
}

.text-box .box-container {
  background-color: #f1f1f1;
  border-radius: 8px 8px 21px 21px;
  padding: 8px;
}

.text-box textarea {
  width: 100%;
  height: 40px;
  resize: none;
  border: 0;
  border-radius: 6px;
  padding: 12px 12px 10px 12px;
  font-size: 13px;
  outline: none;
  caret-color: #0a84ff;
}

.text-box .formatting button {
  width: 30px;
  height: 30px;
  position: relative;
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: transparent;
  border-radius: 50%;
  border: 0;
  outline: none;
}

.text-box .formatting button:hover {
  background-color: #f1f1f1;
}

.text-box .formatting .send {
  width: 30px;
  height: 30px;
  background-color: #0a84ff;
  margin: 0 0 0 auto;
}

.text-box .formatting .send:hover {
  background-color: #026eda;
}

@keyframes ripple {
  0% {
    transform: scale(0);
    opacity: 0.6;
  }

  100% {
    transform: scale(1);
    opacity: 0;
  }
}
</style>
