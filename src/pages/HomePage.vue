<template>
  <q-page class="constrain q-pa-md">
    <div class="row q-col-gutter-md">
      <div class="col-12 col-sm-8">
        <template v-if="!loadingPosts && posts.length">
          <q-card
            v-for="post in posts"
            :key="post.id"
            class="post-card q-mb-md"
            :class="{ 'bg-yellow-2': post.offline }"
            flat
            bordered
          >
            <q-badge
              v-if="post.offline"
              transparent
              align="middle"
              color="orange"
              class="absolute-top-right"
            >
              Post stored offline
            </q-badge>
            <q-item>
              <q-item-section avatar>
                <q-avatar>
                  <img alt="img" :src="post.userPhoto" />
                </q-avatar>
              </q-item-section>

              <q-item-section>
                <q-item-label class="text-bold">{{
                  post.postedBy
                }}</q-item-label>
                <q-item-label caption> {{ post.location }}</q-item-label>
              </q-item-section>
            </q-item>
            <q-img :src="post.imageUrl" alt="image" />
            <q-card-section>
              <div>{{ post.caption }}</div>
              <div class="text-caption text-grey">
                {{ niceDate(post.date) }}
              </div>
              <!-- <div class="text-caption">{{ post.writenPost }}</div> -->
            </q-card-section>

            <q-card-actions>
              <q-btn flat color="dark" label="Share" />
              <q-btn flat color="primary" label="like" />
            </q-card-actions>

            <q-separator />
          </q-card>
        </template>
        <template v-else-if="!loadingPosts && !posts.length">
          <h5 class="text-center text-grey">No posts yet.</h5>
        </template>
        <template v-else>
          <q-card flat bordered>
            <q-item>
              <q-item-section avatar>
                <q-skeleton type="QAvatar" animation="fade" size="40px" />
              </q-item-section>

              <q-item-section>
                <q-item-label>
                  <q-skeleton type="text" animation="fade" />
                </q-item-label>
                <q-item-label caption>
                  <q-skeleton type="text" animation="fade" />
                </q-item-label>
              </q-item-section>
            </q-item>

            <q-skeleton height="200px" square animation="fade" />

            <q-card-section>
              <q-skeleton type="text" class="text-subtitle2" animation="fade" />
              <q-skeleton
                type="text"
                width="50%"
                class="text-subtitle2"
                animation="fade"
              />
            </q-card-section>
          </q-card>
        </template>
      </div>

      <div class="col-4 large-screen-only">
        <q-item class="fixed">
          <q-item-section avatar>
            <q-avatar size="48px">
              <img alt="img" :src="user.photoURL" />
            </q-avatar>
          </q-item-section>

          <q-item-section>
            <q-item-label class="text-bold">{{
              user.displayName
            }}</q-item-label>
            <q-item-label caption> {{ user.email }}</q-item-label>
          </q-item-section>
        </q-item>
      </div>
    </div>
  </q-page>
</template>

<script>
import { date } from "quasar";
import { openDB, deleteDB, wrap, unwrap } from "idb";
import { getAuth, onAuthStateChanged } from "firebase/auth";
import { LocalStorage, SessionStorage } from "quasar";
import { useQuasar } from "quasar";

export default {
  name: "HomePage",

  data() {
    return {
      posts: [],
      loadingPosts: false,
      user: window.user,
    };
  },

  computed: {
    serviceWorkerSupported() {
      if ("serviceWorker" in navigator) return true;
      return false;
    },
  },

  methods: {
    getPosts() {
      this.loadingPosts = true;
      this.$axios
        .get(`${process.env.API}/posts`)
        .then((response) => {
          this.posts = response.data;
          this.loadingPosts = false;
          if (!navigator.onLine) {
            this.getOfflinePosts();
          }
        })
        .catch((err) => {
          this.$q.dialog({
            title: "Error",
            message: "Could not find your data",
          });
          this.loadingPosts = false;
        });
    },
    getOfflinePosts() {
      let db = openDB("workbox-background-sync").then((db) => {
        db.getAll("requests")
          .then((failedRequests) => {
            failedRequests.forEach((failedRequest) => {
              if (failedRequest.queueName == "createPostQueue") {
                let request = new Request(
                  failedRequest.requestData.url,
                  failedRequest.requestData
                );
                request.formData().then((formData) => {
                  let offlinePost = {};
                  offlinePost.id = formData.get("id");
                  offlinePost.caption = formData.get("caption");
                  offlinePost.location = formData.get("location");
                  offlinePost.date = parseInt(formData.get("date"));
                  offlinePost.offline = true;

                  let reader = new FileReader();
                  reader.readAsDataURL(formData.get("file"));
                  reader.onloadend = () => {
                    offlinePost.imageUrl = reader.result;
                    this.posts.unshift(offlinePost);
                  };
                });
              }
            });
          })
          .catch((err) => {
            console.log("Error accessing IndexedDB: ", err);
          });
      });
    },
    listenForOfflinePostUploaded() {
      if (this.serviceWorkerSupported) {
        const channel = new BroadcastChannel("sw-messages");
        channel.addEventListener("message", (event) => {
          console.log("Received", event.data);
          if (event.data.msg == "offline-post-uploaded") {
            let offlinePostCount = this.posts.filter(
              (post) => post.offline == true
            ).length;
            this.posts[offlinePostCount - 1].offline = false;
          }
        });
      }
    },
    niceDate(value) {
      return date.formatDate(value, "MMMM D h:mmA");
    },
  },

  activated() {
    console.log("activated");
    this.getPosts();
  },

  created() {
    // const $q = useQuasar();
    // const User = $q.localStorage.getItem(user);
    // console.log(User);
    // this.user = User;
    // const value = $q.localStorage.getItem(key)
    // const value = $q.localStorage.getItem(key)
    if (!window.user) {
      this.$router.push(`/login`);
    }
    const auth = getAuth();
    onAuthStateChanged(auth, (user) => {
      if (user) {
        // User is signed in, see docs for a list of available properties
        // https://firebase.google.com/docs/reference/js/firebase.User
        const uid = user.uid;
        // ...
      }
    });
    this.listenForOfflinePostUploaded();
  },
};
</script>

<style lang="sass">
.q-card
  .q-img
    min-height: 200px
</style>
