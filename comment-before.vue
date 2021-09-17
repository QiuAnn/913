<template>
  <div class="comments-wrapper" :class="[comments.length > 0 ? 'comments-no-blank' : 'comments-blank']">
    <p v-if="message && !loading">{{ message }}</p>
    <p v-if="comments.length < 1 && !message && !loading && !lineCode" class="text-muted comments-empty-text">
      {{ $t('comments.empty') }}
    </p>

    <div :class="{ empty: comments.length < 1 }" class="ui comments math-code">
      <template v-for="comment in comments">
        <div v-if="!(comment.pull_request_review&&comment.root_node)">
        <!-- <div v-if="true"> -->
          <comment
            :key="comment.id"
            v-if="noteableType !== 'Issue' || (noteableType === 'Issue' && comment.type === 'issue_note')"
            :comments-url="url"
            :comment="comment"
            :noteable-id="noteableId"
            :noteable-type="noteableType"
            :line-code="lineCode"
            :editor-source-url="editorSourceUrl"
            :is-community="isCommunity"
            :line-num="lineNum"
            :comment-args="commentArgs"
            :is-comment-tab="isCommentTab"
            @remove="removeComment"
            @update="updateComment"
            @select="selectCodeComment"
            @reply="replyComment"
          >
            <!-- <slot name="comment-box" :diff-position-id="comment.diff_position_id" /> -->
          </comment>
        </div>
        <div v-else-if="comment.pull_request_review_state!=0" class="code-review-comments" >
          <div class="title d-flex-between">
          <div class="comment-file-avatar d-align-center mb-2 mt-2">
            <user-link :user="comment.author" class="avatar img" no-text avatar />
            <user-link :user="comment.author" class="author name ml-1" />
            <span
              v-if="comment.project_role"
              :class="{
                owner: comment.project_role === '拥有者',
                member: comment.project_role === '成员'
              }"
              class="commenter-role-label"
              v-text="comment.project_role"
            />
            <user-label :user="comment.author" />&nbsp;
            <span class="at ml-05 mr-05">对</span>
            <div class="text">文件进行{{comment.pull_request_review_state === 3?'评审':'评论'}}</div>
            <span
              v-if="comment.pull_request_review_state === 3"
              class="code-review-label assess"
              v-text="'评审'"
            />
          </div>
            <dropdown class="comment-info-right comment-menu-more">
            <div class="iconfont icon-ic-action-more"></div>
            <div class="menu">
              <!-- <a  class="btn-goto item" @click="">
                {{ $t('comments.goto') }}
              </a> -->
              <!-- <a
                v-if="comment.operating && comment.operating.can_modify"
                class="btn-edit item"
                @click=""
              > 
                {{ $t('comments.edit') }}
              </a>-->
              <a v-if="comment.operating && comment.operating.can_delete" class="btn-delete item" @click="remove(comment)">
                {{ $t('comments.delete') }}
              </a>
            </div>
          </dropdown>
         </div>
          <div
            v-highlight
            v-mathjax
            v-issue-catcher
            v-pull-catcher
            v-checkbox="{ canCheckbox: comment.operating.can_modify }"
            :username="comment.author.username"
            :type-id="comment.id"
            class="text note-content markdown-body ml-5"
            type="note"
            v-html="html(comment)"
          />
          <div class="code-review-comment">
            <!-- {{comment.created_at}} -->
            <template v-for="_comment in comment.notes">
              <comment
                :key="_comment.id"
                v-if="noteableType !== 'Issue' || (noteableType === 'Issue' && _comment.type === 'issue_note')"
                :comments-url="url"
                :comment="_comment"
                :noteable-id="noteableId"
                :noteable-type="noteableType"
                :line-code="lineCode"
                :editor-source-url="editorSourceUrl"onRemoveCommentReview 
                :is-community="isCommunity"
                :line-num="lineNum"
                :comment-args="commentArgs"
                :is-comment-tab="isCommentTab"
                :is-code-review = comment.pull_request_review
                @remove="removeComment"
                @update="updateComment"
                @select="selectCodeComment"
                @reply="replyComment"
              >
              </comment>
            </template>
          </div>
        </div>
        
      </template>
      <div v-if="loading" class="ui active inverted dimmer">
        <div class="ui loader" />
      </div>
    </div>
  </div>
</template>

<script>
import Comment from 'gitee-ent/packages/comment'
import CommentsMixin from 'gitee-ent/src/mixins/comments'
import UserLabel from 'gitee-ent/packages/user-label'
import UserLink from 'gitee-ent/packages/user-link'
import Dropdown from 'gitee-ent/packages/dropdown'
import {wrapImgTag } from 'gitee-ent/src/utils/note'
import GiteeModal from 'gitee-ent/packages/gitee-modal'
export default {
  name: 'EntComments',
  components: {
    Comment,
    UserLabel,
    UserLink,
    Dropdown
  },
  mixins: [CommentsMixin],
  props: {
    url: {
      type: String,
      default: ''
    },
    diffContextPath: {
      type: String,
      default: ''
    },
    isDiff: {
      type: Boolean,
      default: false
    },
    existsComments: {
      type: Array
    },
    isCommunity: {
      type: Boolean,
      default: false
    },
    isCommentTab: {
      type: Boolean,
      default: false
    },
    lineNum: {
      type: Object,
      default: () => {}
    },
    range: {
      type: Array
    },
    commentArgs: {
      type: Object,
      default: () => {}
    }
  },
  data() {
    return {
      comments: [],
      commentsCount: -1,
      loading: false,
      loadingTargetId: null,
      message: ''
    }
  },
  watch: {
    url() {
      this.load()
    },
    noteableId() {
      this.load()
    },
    existsComments: {
      handler(value) {
        if (value) {
          this.comments = value
        }
      },
      deep: true,
      immediate: true
    }
  },
  mounted() {
    this.load()
  },
  created() {
    const hub = window.eventHub
    // 对比 conversations.json 接口数据的结构查看该评论组的增删改查
    hub.$on('reload-issue', this.onIssueReload)
    hub.$on('add-comment', this.onAddComment)
    hub.$on('update-comment', this.onUpdateComment)
    hub.$on('update-comment-checkbox', this.onUpdateCommentCheckBox)
    hub.$on('remove-comment', this.onRemoveComment)
    hub.$on('update-resolved-state', this.onUpdateResolvedState)
    this.$once('hook:beforeDestroy', () => {
      hub.$off('reload-issue', this.onIssueReload)
      hub.$off('add-comment', this.onAddComment)
      hub.$off('update-comment', this.onUpdateComment)
      hub.$off('update-comment-checkbox', this.onUpdateCommentCheckBox)
      hub.$off('remove-comment', this.onRemoveComment)
      hub.$off('update-resolved-state', this.onUpdateResolvedState)
    })
  },
  computed: {
    groupComments() {
      // 不再使用前端将数据嵌套
      // computed 有缓存, 引用不变, 内部值变了不会变化
      return this.comments
    },
  },
  methods: {
     html(comment) {
      return wrapImgTag(comment.data.note_html)
    },
    selectIssue(issue) {
      this.$emit('select-issue', issue)
    },
    load() {
      // 代码评论不需要主动加载
      if (this.lineCode) {
        return
      }
      if (this.loadingTargetId === this.noteableId) {
        return
      }
      // 传递的提前获取到的评论不需要再获取
      if (this.existsComments) {
        return
      }
      this.message = ''
      this.comments = []
      this.commentsCount = -1
      if (!this.url || !this.noteableId) {
        return
      }
      const options = {
        params: {
          note: {
            noteable_id: this.noteableId,
            noteable_type: this.noteableType
          }
        }
      }
      this.loading = true
      this.loadingTargetId = this.noteableId
      this.$http
        .get(this.url, options)
        .then(({ data }) => {
          if (this.noteableType === 'Issue') {
            if (this.noteableId !== data.issue_id) {
              return
            }
          } else if (this.noteableType === 'PullRequest' && this.noteableId !== data.pr_id) {
            return
          }
          this.comments = data.list
          this.commentsCount = data.count
          this.$emit('changed', this.comments)
          this.loading = false
          this.loadingTargetId = null
          this.autoScroll()
        })
        .catch(() => {
          this.message = this.$t('comments.load_error')
          this.loading = false
        })
    },
    getStateData(issue) {
      const stateId = issue.state_data.id
      const states = window.getIssueStates(issue.type)
      for (let i = 0; i < states.length; ++i) {
        if (states[i].id === stateId) {
          return states[i]
        }
      }
      return {
        name: '未知',
        color: '#ccc',
        icon: 'iconfont icon-issue'
      }
    },
    getStateIcon(issue) {
      return this.getStateData(issue).icon
    },
    getStateColor(issue) {
      return this.getStateData(issue).color
    },
    findComment(id, callback, notFindCallback) {
      for (let i = 0; i < this.comments.length; ++i) {
        const comment = this.comments[i]
        if (comment.id === id) {
          callback(i, comment)
          return
        }
      }
      notFindCallback && notFindCallback()
    },
    // 移除所有直接或间接关系的子评论 返回被过滤的数据
    removeAllChildNode(currentChild, descendants) {
      if (currentChild) {
        var removedChilds = []
        var restChilds = descendants.filter(item => {
          if (item.parent.id == currentChild.id) {
            removedChilds.push(item)
          }
          return item.parent.id != currentChild.id
        })
        // 从被删除的集合中递归过滤间接关系的子评论
        if (removedChilds && removedChilds.length > 0) {
          removedChilds.forEach((removedItem, index) => {
            restChilds = this.removeAllChildNode(removedItem, restChilds)
            removedChilds.splice(index, 1)
          })
        }
      }
      return restChilds
    },
    // 查找根节点或者子节点
    findNodeIndex(data) {
      let diffPositioId = data.note.diff_position_id
      let commentId = data.note.id
      let isDiff = data.note.data.line_code
      // 是diff评论
      if (isDiff) {
        let diffIndex
        let noteIndex
        let childIndex
        // 先找diff_position_id => noteId => childId
        diffIndex = this.comments.findIndex(item => item.diff_position_id === diffPositioId)
        if (diffIndex > -1) {
          let currentDiff = this.comments[diffIndex]
          currentDiff.notes &&
            (noteIndex = currentDiff.notes.findIndex(item => item.id === commentId || item.id === data.note.root_id)) // !拥有 root_id即是最底层的子评论 兼容两种情况: 找node 或最底层
          if (data.note.root_node) {
            // diff根节点只有两层
            return { type: 'diff', root: true, diffIndex, noteIndex }
          }
          if (noteIndex > -1) {
            let currentChild = currentDiff.notes[noteIndex]
            currentChild &&
              currentChild.descendants &&
              (childIndex = currentChild.descendants.findIndex(item => item.id === commentId))
            noteIndex
          }
        }
        return { type: 'diff', diffIndex, noteIndex, childIndex }
      } else {
        // 普通评论
        let noteIndex
        let childIndex
        noteIndex = this.comments.findIndex(item => item.id === commentId || item.id === data.note.root_id)
        if (data.note.root_node) return { type: 'normal', root: true, noteIndex }
        ~noteIndex &&
          this.comments[noteIndex] &&
          (childIndex = this.comments[noteIndex].descendants.findIndex(item => item.id === commentId))
        return { type: 'normal', noteIndex, childIndex }
      }
    },
    onAddComment(data,isreview=false) {
      if(isreview||data.note.pull_request_review_state!=null){
        this.handlerReview(data,isreview)
        return 
      }
      if (data.noteableId === this.noteableId && data.noteableType === this.noteableType) {
        // if (this.lineCode) {
        //   if (this.lineCode != data.note.data.line_code) {
        //     return
        //   }
        // }
        let isDiff = data.note.data ? data.note.data.line_code : false
        if (data.parentComment && data.parentComment.data) {
          isDiff = isDiff ? isDiff : data.parentComment.data.line_code // 找不到就从父评论找
          // commit 中回复子评论 后端无法给到 line_code
          isDiff && (data.note.data.line_code = data.parentComment.data.line_code)
        }

        if (!data.note.resolved_user) data.note.resolved_user = {}
        // 后端返回的 data中的 parent.id 可能是子评论的id. 在视图上没有子评论回复子评论. 子评论与子评论同级别.
        // 使用 parent.id 查找 notes时 如果未找到可能是 parent.id 是某个子评论的id, 则直接插入

        let parentId = data.note.parent ? data.note.parent.id : ''
        // 如果返回的数据存在 diff_position_id 则是diff评论, 否则是普通评论
        // 更正: comit无法给到diff_position_id, 用 line_code判断
        if (isDiff) {
          //1. 找到diff_if在原评论列表是否存在, 不存在这是添加新diff评论
          //2. 存在则继续找到notes中对应的父评论id, 将此数据插入
          let diffIndex = -1;
            //pr 页面的评论列表会返回 评审的内容，并且在“文件页"参与循环。所以如果是添加普通的评论，应该找的是普通评论的diff_position_id
          diffIndex = this.comments.findIndex(item => {
            return item.diff_position_id === data.note.diff_position_id && item.pull_request_review_state == null
          })
          if (diffIndex > -1) {
            let currentItem = this.comments[diffIndex]
            currentItem.notes ? null : this.$set(currentItem, 'notes', [])
            // 没有parentId, 与diff平级的特殊评论
            if (!parentId && data.note.diff_position) {
              // 添加评论时验证下存在性
              if (currentItem.notes.every(comment => comment.id !== data.note.id)) {
                currentItem.notes.push(data.note)
                this.$set(this.comments, diffIndex, currentItem)
              }
              return
            }
            let noteIndex = currentItem.notes.findIndex(item => item.id === parentId)
            // 找到父note, 插入到该节点下,更新  this.comments
            // descendants ||= []
            if (noteIndex > -1) {
              currentItem.notes[noteIndex].descendants || this.$set(currentItem.notes[noteIndex], 'descendants', [])
              // 添加评论时验证下存在性
              if (currentItem.notes[noteIndex].descendants.every(comment => comment.id !== data.note.id)) {
                currentItem.notes[noteIndex].descendants.push(data.note)
                this.$set(this.comments, diffIndex, currentItem)
              }
            } else {
              // 回复了子评论, parentId 也是子评论
              // 后端返回缺少 root_id
              if (!data.parentComment.root_id) return
              let rootIndex = currentItem.notes.findIndex(item => item.id === data.parentComment.root_id)
              // 添加评论时验证下存在性
              if (
                rootIndex > -1 &&
                currentItem.notes[rootIndex].descendants.every(comment => comment.id !== data.note.id)
              ) {
                currentItem.notes[rootIndex].descendants.push(data.note)
                this.$set(this.comments, diffIndex, currentItem)
              }
            }
          } else {
            // 添加新的diff评论组
            // 添加评论时验证下存在性
            if (this.comments.every(comment => comment.diff_position_id !== data.note.diff_position_id)) {
              // 兼容commit页面 添加评论返回 note_id
              if (data.note.diff_position && !data.note.diff_position.diff_position_id) {
                data.note.diff_position.diff_position_id = data.note.diff_position.note_id
                data.note.diff_position.notes[0].diff_position_id = data.note.diff_position.note_id
              }
              this.comments.push(data.note.diff_position)
            }
          }
        } else {
          // 回复普通评论
          data.note.item_kind = 'note'
          // 如果没有parentComment则是新添加评论,而非回复
          if (data.parentComment) {
            let rootIndex = -1
            // 存在传过来的父评论 则一定能在comments中找到 索引
            // todo: 评论后后端返回的数据缺少 root_id
            // 根节点直接用id查找
            if (data.parentComment.root_node) {
              rootIndex = this.comments.findIndex(item => item.id == data.parentComment.id)
            } else {
              // 不是根节点则是回复了子评论 根据 root_id 查找
              rootIndex = this.comments.findIndex(item => item.id == data.parentComment.root_id)
            }
            if (rootIndex < 0) return
            let rootComment = this.comments[rootIndex]
            rootComment.descendants || this.$set(rootComment, 'descendants', [])
            rootComment.descendants.push(data.note)
            this.$set(this.comments, rootIndex, rootComment)
          } else {
            this.comments.push(data.note)
          }
        }
        this.$emit('changed', this.comments)
        this.scrollToTail()
      }
    },
    handlerReview(data, isreview){
      if (this.isDiff) { //第一次是comments tab页会执行一次，剩下的是files tab执行，所有评审的只要comments 执行一次就好。
        return
      }
      if (data.noteableId === this.noteableId && data.noteableType === this.noteableType) {
        let isDiff = data.note.data ? data.note.data.line_code : false
        if (data.parentComment && data.parentComment.data) {
          isDiff = isDiff ? isDiff : data.parentComment.data.line_code // 找不到就从父评论找
          // commit 中回复子评论 后端无法给到 line_code
          isDiff && (data.note.data.line_code = data.parentComment.data.line_code)
        }

        if (!data.note.resolved_user) data.note.resolved_user = {}
        // 后端返回的 data中的 parent.id 可能是子评论的id. 在视图上没有子评论回复子评论. 子评论与子评论同级别.
        // 使用 parent.id 查找 notes时 如果未找到可能是 parent.id 是某个子评论的id, 则直接插入

        let parentId = ""
        if ((data.note.parent) && (data.note.root_id != data.note.parent.id)) { //评审的
          parentId = data.note.parent ? data.note.parent.id : ''
        }
        //评审都是对代码的评论
        if (isDiff) {
          //先查找 是否已经存在评审状态的评论，存在就只要操作评审的数据就好
          let indexByPending = this.comments.findIndex(item => {
            return item.pull_request_review_state === data.note.pull_request_review_state && item.id == data.note.ancestor_ids[0] //评审状态的 对应的某种评论（待提交0/评审3/通过2）
          })
          let _comments = this.comments
          if (indexByPending > -1) {
            _comments = this.comments[indexByPending].notes
          }
          //1. 找到diff_if在原评论列表是否存在, 不存在这是添加新diff评论
          //2. 存在则继续找到notes中对应的父评论id, 将此数据插入
          let diffIndex = _comments.findIndex(item => {
            return item.diff_position_id === data.note.diff_position_id
          })
          if (diffIndex > -1) {
            let currentItem = _comments[diffIndex]
            currentItem.notes ? null : this.$set(currentItem, 'notes', [])
            // 没有parentId, 与diff平级的特殊评论
            if (!parentId && data.note.diff_position) {
              // 添加评论时验证下存在性
              if (currentItem.notes.every(comment => comment.id !== data.note.id)) {
                currentItem.notes.push(data.note)
                this.$set(_comments, diffIndex, currentItem)
              }
              return
            }

            let noteIndex = currentItem.notes.findIndex(item => item.id === parentId)
            // 找到父note, 插入到该节点下,更新  this.comments
            // descendants ||= []
            if (noteIndex > -1) {
              currentItem.notes[noteIndex].descendants || this.$set(currentItem.notes[noteIndex], 'descendants', [])
              // 添加评论时验证下存在性
              if (currentItem.notes[noteIndex].descendants.every(comment => comment.id !== data.note.id)) {
                currentItem.notes[noteIndex].descendants.push(data.note)
                this.$set(_comments, diffIndex, currentItem)
              }
            } else {
              // 回复了子评论, parentId 也是子评论
              // 后端返回缺少 root_id
              if (!data.parentComment.root_id) return
              let rootIndex = currentItem.notes.findIndex(item => {
                return item.id === data.parentComment.ancestor_ids[1]
              })
              // 添加评论时验证下存在性
              if (
                rootIndex > -1 &&
                currentItem.notes[rootIndex].descendants.every(comment => comment.id !== data.note.id)
              ) {
                currentItem.notes[rootIndex].descendants.push(data.note)
                this.$set(_comments, diffIndex, currentItem)
              }
            }
          } else {
            //不存在 就添加一条
            if (_comments.every(comment => comment.diff_position_id !== data.note.diff_position_id)) {
              _comments.push(data.note.diff_position)
            }
          }
        }
        this.$emit('changed', this.comments, isreview)
        this.scrollToTail()
      }
    },
    onUpdateComment(data) {
      if (data.note.pull_request_review_state != null) {
        this.onUpdateCommentReview(data)
        return
      }
      // 编辑请求 后端也应该返回 root_id
      if (data.noteableId === this.noteableId && data.noteableType === this.noteableType) {
        let oIndexInfo = this.findNodeIndex(data)
        if (oIndexInfo.type == 'diff') {
          let { diffIndex, noteIndex, childIndex, root } = oIndexInfo
          if (diffIndex < 0 || noteIndex < 0 || !this.comments[diffIndex].notes) return
          // 替换diff评论根节点
          // root && this.comments[diffIndex].notes.splice(noteIndex, 1, data.note)
          root && (this.comments[diffIndex].notes[noteIndex].data = data.note.data)
          // 替换diff评论子节点
          ~childIndex && !root && this.comments[diffIndex].notes[noteIndex].descendants.splice(childIndex, 1, data.note)
          this.$set(this.comments, diffIndex, this.comments[diffIndex])
        } else {
          let { noteIndex, childIndex, root } = oIndexInfo
          if (noteIndex < 0) return
          // 替换普通评论根节点
          // root && this.comments.splice(noteIndex, 1, data.note)
          root && (this.comments[noteIndex].data = data.note.data)
          // 替换普通评论子节点
          this.comments[noteIndex] &&
            this.comments[noteIndex].descendants &&
            ~childIndex &&
            !root &&
            this.comments[noteIndex].descendants.splice(childIndex, 1, data.note)
          this.$set(this.comments, noteIndex, this.comments[noteIndex])
        }
      }
      this.$emit('changed', this.comments)
    },
    onUpdateCommentCheckBox(data) {
      // todo ？
      const n = data
      this.findComment(n.note.id, i => {
        n.note.data.note_html = this.comments[i].data.note_html //用于解决一次更新多个checkbox时出现勾选效果刷新重复问题
        this.comments.splice(i, 1, n.note)
      })
    },
    // 删除子评论 应该继续删除回复这个子评论的评论(也是子评论)
    onRemoveComment(data) {
      if (data.note.pull_request_review_state != null) {
        this.onRemoveCommentReview(data)
        return
      }

      if (data.noteableId === this.noteableId && data.noteableType === this.noteableType) {
        // 删除主评论 ,子评论都会被删除
        let oIndexInfo = this.findNodeIndex(data)
        if (oIndexInfo.type == 'diff') {
          let { diffIndex, noteIndex, childIndex, root } = oIndexInfo
          if (diffIndex < 0 || noteIndex < 0 || !this.comments[diffIndex].notes) return
          // 删除diff评论根节点
          root && this.comments[diffIndex].notes.splice(noteIndex, 1)
          // 删除diff评论子节点
          let currentChild =
            ~childIndex && !root ? this.comments[diffIndex].notes[noteIndex].descendants[childIndex] : null
          currentChild && this.comments[diffIndex].notes[noteIndex].descendants.splice(childIndex, 1)
          // 删除子评论时候 继续删除以 以这条评论为 parentId的评论, 子评论数据和视图上都是平级的 但是存在子父关系.
          if (currentChild) {
            this.comments[diffIndex].notes[noteIndex].descendants = this.removeAllChildNode(
              currentChild,
              this.comments[diffIndex].notes[noteIndex].descendants
            )
          }
          // notes为空 删除自身
          if (this.comments[diffIndex].notes.length === 0) {
            this.comments.splice(diffIndex, 1)
          } else {
            this.$set(this.comments, diffIndex, this.comments[diffIndex])
          }
        } else {
          let { noteIndex, childIndex, root } = oIndexInfo
          if (noteIndex < 0) return
          // 删除普通评论根节点
          if (root) {
            this.comments.splice(noteIndex, 1)
            return
          }
          // 删除普通评论子节点
          var normalChild =
            this.comments[noteIndex] && this.comments[noteIndex].descendants && ~childIndex && !root
              ? this.comments[noteIndex].descendants[childIndex]
              : null
          normalChild && this.comments[noteIndex].descendants.splice(childIndex, 1)
          // 删除子评论时候 继续删除以 以这条评论为 parentId的评论
          if (normalChild) {
            this.comments[noteIndex].descendants = this.removeAllChildNode(
              normalChild,
              this.comments[noteIndex].descendants
            )
          }

          this.$set(this.comments, noteIndex, this.comments[noteIndex])
        }
      }
      this.$emit('changed', this.comments)
    },
     // 查找根节点或者子节点d
    findNodeIndexReview(data) {
      let diffPositioId = data.note.diff_position_id
      let commentId = data.note.id
      let isDiff = data.note.data.line_code
      // 是diff评论
      if (isDiff) {
        let diffIndex
        let noteIndex
        let childIndex
        let indexByPending = this.comments.findIndex(item => {
          //评审状态的 对应的某种评论（待提交0/评审3/通过2）
          //这里是二级，所以ancestor_ids的值就是id
          return item.pull_request_review_state === data.note.pull_request_review_state && item.id == data.note.ancestor_ids[0]
        })
        let _comments = this.comments
        if (indexByPending > -1) {
          _comments = this.comments[indexByPending].notes
        }
        diffIndex = _comments.findIndex(item => item.diff_position_id === diffPositioId)
        if (diffIndex > -1) {
          let currentDiff = _comments[diffIndex]
          currentDiff.notes &&
            (noteIndex = currentDiff.notes.findIndex(item => item.id === commentId || item.id === data.note.ancestor_ids[1])) // !拥有 root_id即是最底层的子评论 兼容两种情况: 找node 或最底层
          if (data.note.ancestor_ids && data.note.ancestor_ids.length == 1) { //就是评审的第二层（相当于普通评论的root_node,即顶级）
            // diff根节点只有两层
            return { type: 'diff', root: true, diffIndex, noteIndex, indexByPending }
          }
          if (noteIndex > -1) {
            let currentChild = currentDiff.notes[noteIndex]
            currentChild &&
              currentChild.descendants &&
              (childIndex = currentChild.descendants.findIndex(item => item.id === commentId))
            noteIndex
          }
        }
        return { type: 'diff', diffIndex, noteIndex, childIndex, indexByPending }
      }
    },
    onRemoveCommentReview(data) {
      // 编辑请求 后端也应该返回 root_id
      if (this.isDiff) {
        return
      }
      if (data.noteableId === this.noteableId && data.noteableType === this.noteableType) {
        if (data.note.root_node) { //顶级
          let _index = this.comments.findIndex(item => item.id === data.note.id)
          if (_index > -1) {
            this.comments.splice(_index, 1)
          }
        } else {
          // 删除主评论 ,子评论都会被删除
          let oIndexInfo = this.findNodeIndexReview(data)
          if (oIndexInfo.type == 'diff') {
            let { diffIndex, noteIndex, childIndex, root, indexByPending } = oIndexInfo
            let __comments = this.comments;
            if (indexByPending > -1) {
              __comments = this.comments[indexByPending].notes
            }

            if (diffIndex < 0 || noteIndex < 0 || !__comments[diffIndex].notes) return
            // 删除diff评论根节点
            root && __comments[diffIndex].notes.splice(noteIndex, 1)
            // 删除diff评论子节点
            let currentChild =
              ~childIndex && !root ? __comments[diffIndex].notes[noteIndex].descendants[childIndex] : null
            currentChild && __comments[diffIndex].notes[noteIndex].descendants.splice(childIndex, 1)
            // 删除子评论时候 继续删除以 以这条评论为 parentId的评论, 子评论数据和视图上都是平级的 但是存在子父关系.
            if (currentChild) {
              __comments[diffIndex].notes[noteIndex].descendants = this.removeAllChildNode(
                currentChild,
                __comments[diffIndex].notes[noteIndex].descendants
              )
            }
            // notes为空 删除自身
            if (__comments[diffIndex].notes.length === 0) {
              __comments.splice(diffIndex, 1)
            } else {
              this.$set(__comments, diffIndex, __comments[diffIndex])
            }
          }
        }
      }
      this.$emit('changed', this.comments)
    },
    onUpdateCommentReview(data) {
      // 编辑请求 后端也应该返回 root_id
      if (this.isDiff) {
        return
      }
      if (data.noteableId === this.noteableId && data.noteableType === this.noteableType) {
        let oIndexInfo = this.findNodeIndexReview(data)
        if (oIndexInfo.type == 'diff') {
          let { diffIndex, noteIndex, childIndex, root, indexByPending } = oIndexInfo
          let __comments = this.comments
          if (indexByPending > -1) {
            __comments = this.comments[indexByPending].notes
          }
          if (diffIndex < 0 || noteIndex < 0 || !__comments[diffIndex].notes) return
          // 替换diff评论根节点
          root && (__comments[diffIndex].notes[noteIndex].data = data.note.data)
          // 替换diff评论子节点
          ~childIndex && !root && __comments[diffIndex].notes[noteIndex].descendants.splice(childIndex, 1, data.note)
          this.$set(__comments, diffIndex, __comments[diffIndex])

        }
      }
      this.$emit('changed', this.comments)
    },
    onIssueReload(issue) {
      if (this.noteableId === issue.id && this.noteableType === 'Issue') {
        this.load()
      }
    },
    onUpdateResolvedState(data) {
      let index = this.comments.findIndex(item => item.diff_position_id == data.note.diff_position_id)
      if (index < 0) return
      let current = this.comments[index]
      current.resolved_state = data.state
      current.resolved_user = data.resolved_user
      this.$set(this.comments, index, current)
    },
    remove(data) {
      GiteeModal.confirm(null, this.$t('comments.confirm_delete'), () => {
        this.$http
          .delete(data.update_path)
          .then(() => {
            this.removeComment(data)
          })
          .catch(() => {
             GiteeModal.alert(this.$t('comments.delete_failed'))
          })
      })
    },
  }
}
</script>
