<template>
  <div v-if="!(comment.pull_request_review&&comment.root_node)">
    <div
      class="the-comment-group"
      :class="[isDiff ? 'diff' : 'normal']"
      v-if="isDiff ? comment.notes && comment.notes.length : true"
    >
      <comment-normal
        v-if="!isDiff"
        :comment="comment"
        :isDiff="isDiff"
        :key="comment.created_at"
        :idkey="idkey"
        :lineCode="lineCode"
        :isCommentTab="isCommentTab"
        :range="range"
        :commentArgs="commentArgs"
        @remove="$emit('remove', $event)"
        @update="$emit('update', $event)"
        @select="$emit('select', $event)"
        @reply="$emit('reply', $event)"
      >
      </comment-normal>
      <template v-else>
        <div class="comment-file-avatar d-align-center mb-2" v-if="comment.diff_position_id && isCommentTab && !comment.pull_request_review">
          <user-link :user="comment.author" class="avatar img" no-text avatar />
          <user-link :user="comment.author" class="author name ml-1" />
          <span
            v-if="isCommunity && comment.project_role"
            :class="{
              owner: comment.project_role === '拥有者',
              member: comment.project_role === '成员'
            }"
            class="commenter-role-label"
            v-text="comment.project_role"
          />
          <user-label :user="comment.author" />&nbsp;
          <span class="at ml-05 mr-05">对</span>
          <div class="text">文件进行评论</div>
        </div>
        <comment-diff
         v-if="!(isCommentTab&&comment.pull_request_review_state==0)"
          :comment="comment"
          :isDiff="isDiff"
          :key="comment.created_at + comment.diff_position_id"
          :diffContext="diffContext"
          :idkey="idkey"
          :lineCode="lineCode"
          :isCommentTab="isCommentTab"
          :range="range"
          :commentArgs="commentArgs"
          @remove="$emit('remove', $event)"
          @update="$emit('update', $event)"
          @select="$emit('select', $event)"
          @reply="$emit('reply', $event)"
        >
        </comment-diff>
      </template>
    </div>
  </div>
  <div v-else-if="comment.pull_request_review_state!=0">
    <comment-normal
      v-if="!isDiff"
      :comment="comment"
      :isDiff="isDiff"
      :key="comment.created_at"
      :idkey="idkey"
      :lineCode="lineCode"
      :isCommentTab="isCommentTab"
      :range="range"
      :commentArgs="commentArgs"
      @remove="$emit('remove', $event)"
      @update="$emit('update', $event)"
      @select="$emit('select', $event)"
      @reply="$emit('reply', $event)"
      >
    </comment-normal>
    <template v-for="_comment in comment.notes">
      <comment
        :key="_comment.id"
        :comments-url="commentsUrl"
        :comment="_comment"
        :noteable-id="noteableId"
        :noteable-type="noteableType"
        :line-code="lineCode"
        :editor-source-url="editorSourceUrl"
        :is-community="isCommunity"
        :line-num="lineNum"
        :comment-args="commentArgs"
        :is-comment-tab="isCommentTab"
        :is-code-review = comment.pull_request_review
        @remove="$emit('remove', $event)"
        @update="$emit('update', $event)"
        @select="$emit('select', $event)"
        @reply="$emit('reply', $event)"
      >
      </comment>
    </template>
  </div>
</template>

<script>
import commentNormal from './comment-normal'
import commentDiff from './comment-diff'
import UserLabel from 'gitee-ent/packages/user-label'
import UserLink from 'gitee-ent/packages/user-link'
export default {
  name: 'comment',
  components: {
    commentNormal,
    commentDiff,
    UserLabel,
    UserLink
  },
  props: {
    commentsUrl: {
      type: String,
      default: ''
    },
    noteableId: {
      type: [Number, String],
      default: 0
    },
    noteableType: {
      type: String,
      default: 'issue'
    },
    isCommentTab: {
      type: Boolean,
      default: false
    },
    comment: {
      type: Object,
      required: true
    },
    editorSourceUrl: {
      type: String,
      default: ''
    },
    idkey: {
      type: String,
      default: 'comment'
    },
    lineCode: {
      type: String,
      default: ''
    },
    isCommunity: {
      type: Boolean,
      default: false
    },
    // 单行评论时需传递的行号
    // 有此参数是文件中的评论
    lineNum: {
      type: Object,
      default: () => {}
    },
    // 多行评论时的范围
    range: {
      type: Array
    },
    commentArgs: {
      type: Object,
      default: () => {}
    },
  },
  provide() {
    const buildLineObj = (line) => ({
      line_num: line.new_line,
      // 后端返回的 diff patch 会在行首带有 +/-/space 符号，这里需要去掉
      content: line.text.slice(1)
    })

    return {
      injectMain: this,
      // 如果存在评论的情况下，这里的 getSuggestionContent() 将会覆盖 line-comments 中的 getSuggestionContent()
      getSuggestionContent: () => {
        const position = this.diffContext.position
        if (!position) return null
        if (position.start_line_code) {
          const selectedLines = this.diffContext.content.filter(f => f.type !== 'old' && f.type !== 'match')
          if (!selectedLines.length) return null

          return {
            filePath: position.new_path,
            lineCode: position.start_line_code,
            diffHunk: selectedLines.map(buildLineObj)
          }
        } else {
          const line = this.diffContext.content.last()
          if (line.type === 'old' || line.type === 'match') return null
          return {
            filePath: position.new_path,
            lineCode: position.line_code,
            diffHunk: [buildLineObj(line)]
          }
        }
      }
    }
  },
  data() {
    return {}
  },
  computed: {
    isDiff() {
      return this.comment.data
        ? this.comment.data.line_code
        : false || this.comment.position
        ? this.comment.position.line_code
        : false
    },
    // 现在不需要 diffContexts.filter(f => f.diff_position_id === comment.diff_position_id).first()
    // 直接在评论中获取, 评论分item_kind 为 'diff_position' 和  note
    // 评论中的diff, 文件中没有
    diffContext() {
      if (this.isDiff) return this.comment
      return {}
    }
  }
}
</script>