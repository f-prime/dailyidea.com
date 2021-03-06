<template>
  <div class="comments-part">
    <div class="comments-part__header text-center">
      <strong class="muted">Comments</strong>
    </div>
    <v-container ref="scroller" class="comments-part__container">
      <div v-if="commentList.length" ref="commentsCol" class="">
        <div
          v-if="idea.commentsCount > commentList.length && !deletingComment"
          class="loadComments"
          @click="loadComments"
        >
          <v-btn small :loading="loadingMore">Load More...</v-btn>
        </div>

        <idea-comments-comment
          v-for="comment in commentList"
          :key="comment.commentId"
          :comment="comment"
          @onDeleteComment="onDeleteComment"
        ></idea-comments-comment>
      </div>
      <v-row v-else class="empty" align="center">
        <v-col class="muted">
          <p>
            <v-icon>mdi-comment-plus-outline</v-icon>
          </p>
          <p>No comments yet.</p>
          <p>Be the first!</p>
        </v-col>
      </v-row>
    </v-container>
    <div class="comments-part__input-container">
      <v-text-field
        v-model="newCommentText"
        label="Add a comment..."
        single-line
        flat
        hide-details
        solo
        @keydown.enter="onAddCommentAttempt"
      >
        <template v-slot:append>
          <v-slide-y-transition hide-on-leave>
            <v-icon
              v-if="readyForSend && !showAddCommentLoader"
              small
              class="color-primary"
              @click="onAddCommentAttempt"
              >fa-paper-plane
            </v-icon>
          </v-slide-y-transition>
          <v-icon
            v-if="showAddCommentLoader"
            small
            class="color-primary"
            @click="onAddCommentAttempt"
            >fas fa-circle-notch fa-spin flag-icon
          </v-icon>
        </template>
      </v-text-field>
    </div>
    <simple-dialog-popup ref="simpleDialogPopup"></simple-dialog-popup>
    <on-first-comment-instantiated-dialog
      ref="onFirstCommentInstantiatedDialog"
    ></on-first-comment-instantiated-dialog>
    <on-un-auth-action-ask-email-dialog
      ref="onUnAuthActionAskEmailDialog"
    ></on-un-auth-action-ask-email-dialog>
    <on-un-auth-action-ask-name-dialog
      ref="onUnAuthActionAskNameDialog"
    ></on-un-auth-action-ask-name-dialog>
  </div>
</template>

<script>
import nanoid from 'nanoid'
import { graphqlOperation } from '@aws-amplify/api'
import IdeaCommentsComment from './IdeaCommentsComment'
import addComment from '~/graphql/mutations/addComment'
import getComments from '~/graphql/query/getComments'
import deleteComment from '~/graphql/mutations/deleteComment'
import simpleDialogPopup from '~/components/dialogs/simpleDialogPopup'
import checkEmailBelongsToExistingUser from '@/graphql/query/checkEmailBelongsToExistingUser'
import onUnAuthActionAskEmailDialog from '@/components/ideaDetail/onUnAuthActionAskEmailDialog'
import onUnAuthActionAskNameDialog from '@/components/ideaDetail/onUnAuthActionAskNameDialog'
import addIdeaTemporaryComment from '@/graphql/mutations/addIdeaTemporaryComment'
import getIdeaTemporaryComment from '@/graphql/query/getIdeaTemporaryComment'
import OnFirstCommentInstantiatedDialog from '@/components/ideaDetail/onFirstCommentInstantiatedDialog'
import setWasWelcomed from '@/graphql/mutations/setWasWelcomed'

const COMMENTS_COUNT = 25

export default {
  name: 'IdeaComments',
  components: {
    OnFirstCommentInstantiatedDialog,
    IdeaCommentsComment,
    simpleDialogPopup,
    onUnAuthActionAskEmailDialog,
    onUnAuthActionAskNameDialog
  },
  props: {
    idea: {
      type: Object,
      required: true
    }
  },
  data() {
    return {
      tmpComment: {},
      newCommentText: '',
      showAddCommentLoader: false,
      commentList: [],
      nextToken: null,
      loadingMore: false,
      deletingComment: false,
      temporaryCommentId: undefined
    }
  },
  computed: {
    readyForSend() {
      return this.newCommentText.length > 0
    },
    isAuthenticated() {
      return this.$store.getters['userData/isAuthenticated']
    }
  },
  mounted() {
    this.doInitialCommentsLoading()
  },
  methods: {
    scrollToBottom() {
      this.$nextTick(() => {
        this.$refs.scroller.scrollTop = this.$refs.scroller.scrollHeight
      })
    },
    doInitialCommentsLoading() {
      if (this.idea.commentsCount > 0) {
        const fakeCommentsCount = Math.min(
          COMMENTS_COUNT,
          this.idea.commentsCount
        )
        const fakeComments = []
        for (let i = 0; i < fakeCommentsCount; i++) {
          fakeComments.push({
            fake: true,
            commentId: nanoid()
          })
        }
        this.commentList = fakeComments
        this.scrollToBottom()
      }
      let additionalAction

      if (this.$route.query.aa) {
        additionalAction = this.$route.query.aa
        if (additionalAction === 'itc') {
          // instantiate temporary comment
          this.temporaryCommentId = this.$route.query.tci
          this.commentList.push({
            fake: true,
            temporary: true,
            commentId: this.temporaryCommentId
          })
          this.getTemporaryComment(this.temporaryCommentId)
        }
      }
      this.loadComments()
    },
    async getTemporaryComment(commentId) {
      const result = await this.$amplifyApi.graphql({
        query: getIdeaTemporaryComment,
        variables: {
          commentId
        },
        authMode: 'API_KEY'
      })
      if (result.data.getIdeaTemporaryComment.result.ok) {
        const temporaryComment = result.data.getIdeaTemporaryComment.comment
        temporaryComment.temporary = true
        temporaryComment.userName = this.$store.getters['userData/userName']
        this.commentList.splice(this.commentList.length - 1, 1)
        this.commentList.push(temporaryComment)
        this.sendComment(temporaryComment.body, true)
        this.scrollToBottom()
      } else {
        this.commentList.splice(this.commentList.length - 1, 1)
        this.$router.replace({ query: null })
      }
    },
    async onDeleteComment(comment) {
      const confirmed = await this.$refs.simpleDialogPopup.show(
        'Delete Comment',
        'Are you sure you want to delete this comment?',
        'Yes, Delete',
        'Cancel',
        require('~/assets/images/dialogs/undraw_throw_away_ldjd.svg')
      )
      if (!confirmed) {
        return
      }
      this.deletingComment = true
      this.$store.commit('layoutState/showProgressBar')
      try {
        this.commentList = this.commentList.filter(
          c => c.commentId !== comment.commentId
        )
        await this.$amplifyApi.graphql(
          graphqlOperation(deleteComment, {
            // body: body,
            // userId: this.$store.getters['cognito/userSub'],
            ideaId: this.idea.ideaId,
            ideaName: this.idea.ideaName,
            ideaOwnerId: this.idea.userId,
            commentId: comment.commentId
          })
        )
        // remove comment form comment list array
        this.idea.commentsCount -= 1
        this.$emit('onNotification', {
          type: 'success',
          message: 'Comment Deleted!'
        })
      } catch (err) {
        this.$emit('onNotification', {
          type: 'error',
          message: "Can't Delete Comment!"
        })
      }
      this.deletingComment = false
      this.$store.commit('layoutState/hideProgressBar')
    },
    onAddCommentAttempt() {
      const commentText = this.newCommentText
      this.newCommentText = ''
      this.showAddCommentLoader = true
      const isAuthenticated = this.$store.getters['userData/isAuthenticated']
      if (isAuthenticated) {
        this.sendComment(commentText)
      } else {
        this.appendFakeCommentAndEncourageToRegisterOrSignUp(commentText)
      }
    },
    checkEmailBelongsToExistingUser(email) {
      return this.$amplifyApi.graphql({
        query: checkEmailBelongsToExistingUser,
        variables: {
          email
        },
        authMode: 'API_KEY'
      })
    },
    addTemporaryFakeComment(text) {
      this.tmpComment = { userName: 'Me', body: text, temporary: true }
      this.commentList.push(this.tmpComment)
      this.scrollToBottom()
    },
    removeTemporaryFakeComment() {
      this.commentList.splice(this.commentList.indexOf(this.tmpComment), 1)
    },
    processCommentInstantiation() {
      const wasWelcomed = this.$store.getters['userData/wasWelcomed']
      if (wasWelcomed) {
        this.$refs.simpleDialogPopup.show(
          'Welcome back!',
          'Thanks for posting that comment!',
          undefined,
          null
        )
      } else {
        this.$refs.onFirstCommentInstantiatedDialog.show()
        this.$amplifyApi.graphql(
          graphqlOperation(setWasWelcomed, {
            userId: this.$store.getters['userData/userId']
          })
        )
      }
      this.$router.replace({ query: null })
    },
    async createTemporaryCommentInDB(userId, commentText) {
      const res = await this.$amplifyApi.graphql({
        query: addIdeaTemporaryComment,
        variables: {
          userId,
          body: commentText,
          ideaId: this.idea.ideaId,
          ideaName: this.idea.title,
          ideaOwnerId: this.idea.userId
        },
        authMode: 'API_KEY'
      })
      return res.data.addIdeaTemporaryComment.comment
    },
    async requestAuthAndProcessComment(userId, email, name, commentText) {
      this.$store.commit('layoutState/showProgressBar')
      const comment = await this.createTemporaryCommentInDB(userId, commentText)
      await this.$amplifyApi.post('RequestLogin', '', {
        body: { email, commentId: comment.commentId }
      })
      this.$store.commit('layoutState/hideProgressBar')
      this.$refs.simpleDialogPopup.show(
        'Welcome back!',
        "It looks like you weren't signed in. We just sent you a verification email. Please check your inbox and click on the link and we'll post your comment ASAP.",
        undefined,
        null
      )
    },
    async registerUserAndProcessComment(email, name, commentText) {
      try {
        this.$store.commit('layoutState/showProgressBar')
        const res = await this.$store.dispatch('cognito/registerUser', {
          username: email,
          password: nanoid(),
          attributes: {
            name
          }
        })
        const userId = res.userSub
        const comment = await this.createTemporaryCommentInDB(
          userId,
          commentText
        )
        await this.$amplifyApi.post('RequestLogin', '', {
          body: { email, commentId: comment.commentId }
        })
        this.$store.commit('layoutState/hideProgressBar')
        this.$refs.simpleDialogPopup.show(
          'Thanks!',
          "We just sent you an email to confirm that you're a real person :) Please check your inbox then click on the link and we'll post your comment ASAP.",
          'OK',
          null
        )
      } catch (e) {
        this.$store.commit('layoutState/hideProgressBar')
      }
    },
    async appendFakeCommentAndEncourageToRegisterOrSignUp(commentText) {
      this.showAddCommentLoader = false
      this.addTemporaryFakeComment(commentText)
      try {
        let email = await this.$refs.onUnAuthActionAskEmailDialog.show(
          'Introduce yourself?',
          'Before we post this for everyone to see, can you please confirm your email address?',
          'ok',
          'delete comment'
        )
        email = email.toLowerCase()
        this.$store.commit('layoutState/showProgressBar')
        const result = await this.checkEmailBelongsToExistingUser(email)
        const belongsToExistingUser =
          result.data.checkEmailBelongsToExistingUser.belongsToExistingUser
        this.$store.commit('layoutState/hideProgressBar')
        if (belongsToExistingUser) {
          const userId = result.data.checkEmailBelongsToExistingUser.userId
          this.requestAuthAndProcessComment(userId, email, name, commentText)
        } else {
          const name = await this.$refs.onUnAuthActionAskNameDialog.show(
            'Nice to meet you!',
            'OK, thanks! And what can we call you?',
            'ok',
            'delete comment'
          )
          this.registerUserAndProcessComment(email, name, commentText)
        }
      } catch (e) {
        this.removeTemporaryFakeComment()
        this.$store.commit('layoutState/hideProgressBar')
      }
    },
    async sendComment(commentText, instantiation = false) {
      try {
        const result = await this.$amplifyApi.graphql(
          graphqlOperation(addComment, {
            body: commentText,
            ideaId: this.idea.ideaId,
            ideaOwnerId: this.idea.userId,
            userName: this.$store.getters['userData/userName'],
            userSlug: this.$store.getters['userData/slug']
          })
        )
        const newComment = result.data.addComment.comment
        if (instantiation) {
          newComment.temporary = true
          this.$set(this.commentList, this.commentList.length - 1, newComment)
          this.$nextTick(() => {
            this.$set(
              this.commentList[this.commentList.length - 1],
              'temporary',
              false
            )
          })
          this.idea.commentsCount += 1
          setTimeout(() => {
            this.processCommentInstantiation()
          }, 2000)
          this.scrollToBottom()
          return
        } else {
          this.commentList.push(newComment)
          this.idea.commentsCount += 1
          this.scrollToBottom()
        }
        // this.fetchCommentList()
        this.showAddCommentLoader = false

        this.$emit('onNotification', {
          type: 'success',
          message: 'Comment Added!'
        })
      } catch (err) {
        this.$emit('onNotification', {
          type: 'error',
          message: "Can't add Comment. Please reload page and try again."
        })
        this.showAddCommentLoader = false
        this.newCommentText = commentText
      }
    },
    async loadComments() {
      this.loadingMore = true
      try {
        const result = await this.$amplifyApi.graphql({
          query: getComments,
          variables: {
            ideaId: this.$route.params.ideaId,
            limit: COMMENTS_COUNT,
            nextToken: this.nextToken
          },
          authMode: this.$store.getters['userData/isAuthenticated']
            ? undefined
            : 'API_KEY'
        })
        if (this.nextToken) {
          const lastEl = this.$refs.commentsCol.children[1]
          for (const comment of result.data.getComments.items) {
            this.commentList.unshift(comment)
          }
          this.$nextTick(() => {
            lastEl.scrollIntoView()
            this.$refs.scroller.scrollTop -= 55
          })
        } else {
          this.commentList = this.commentList.filter(
            c => c.commentId === this.temporaryCommentId
          ) // remove comments placeholder but not temporary comment - it is loaded separately
          for (const comment of result.data.getComments.items) {
            this.commentList.unshift(comment)
          }
          this.scrollToBottom()
        }

        this.nextToken = result.data.getComments.nextToken
      } catch (e) {
        this.$emit('onNotification', {
          type: 'error',
          message: "Can't load Comments. Please reload page and try again."
        })
        this.commentList = []
      }
      this.loadingMore = false
    }
  }
}
</script>

<style scoped lang="scss">
@import '~assets/style/common';
$light-grey: #f3f0f4;
$counters-font-size: 18px;

.likes-counter {
  display: inline-block;

  &__image-container {
    display: inline-block;
    vertical-align: bottom;
    min-width: $counters-font-size + 7px;
    text-align: center;
    min-height: $counters-font-size + 5px;
  }

  &__image {
    display: inline-block;
    vertical-align: bottom;
    height: $counters-font-size + 1px;

    &.on {
      height: $counters-font-size + 5px;
    }
  }

  &__label {
    margin-left: 5px;
    display: inline-block;
    vertical-align: bottom;
    height: $counters-font-size;
    line-height: $counters-font-size;
    font-size: $counters-font-size;
  }
}

.comments-counter {
  &__image {
    display: inline-block;
    vertical-align: bottom;
    height: $counters-font-size;
  }

  &__label {
    margin-left: 8px;
    display: inline-block;
    vertical-align: bottom;
    height: $counters-font-size;
    line-height: $counters-font-size;
    font-size: $counters-font-size;
  }
}

.comments-part {
  @media (min-width: $screen-md-min) {
    height: calc(100vh - 88px - 24px);
    /* i hate these height calculations. right now this is a mess. i added 24 here because that's how much padding there is */
  }

  overflow: hidden;

  &__header {
    padding: 15px;
    background-color: $light-grey;
  }

  &__container {
    background-color: $light-grey;
    @media (min-width: $screen-md-min) {
      height: calc(100vh - 64px - 12px - 54px - 54px - 10px - 24px);
      /*64 desktop header height 12 main content area padding (below header) 54 comment header 54 comment input*/
      /* why the extra 10 at the end? I don't know! */
    }
    overflow: auto;
    /*height: 100%;*/

    display: block;
  }
  .loadComments {
    text-align: center;
    padding: 5px 0;
    cursor: pointer;
    /* legacy */
  }
  .empty {
    height: 100%;
    text-align: center;
  }

  &__input-container {
    border-radius: 0;
    border: 1px solid $light-grey;
  }
}
</style>
