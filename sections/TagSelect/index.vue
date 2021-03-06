<template>
  <a-popover
    destroyTooltipOnHide
    placement="bottomLeft"
    trigger="click"
    v-model="visible"
    overlayClassName="tag-overlay"
    @visibleChange="handleVisibleChange">
    <template v-if="$scopedSlots.trigger">
      <slot name="trigger" :loading="loading" />
    </template>
    <template v-else>
      <a-button :loading="loading">{{ buttonText }}</a-button>
    </template>
    <template slot="content">
      <div class="tag-wrap" ref="tag-wrap">
        <ul class="tag-list" v-if="filterWithoutUserMeta">
          <li
            class="tag-item"
            :class="{ checked: checkedKeys.includes(withoutUserMetaKey) && value[withoutUserMetaKey][0] === true }"
            @click="handleKeyClick(withoutUserMetaKey, true)">
            <div class="title d-flex align-items-center">
              <div class="flex-fill mr-4 text-truncate">无本地标签资源</div>
              <a-icon class="check-icon" type="check" />
            </div>
          </li>
        </ul>
        <ul class="tag-list" v-if="showUserTags">
          <li class="tag-tip">本地标签</li>
          <li
            class="tag-item"
            v-for="item of userTags"
            :key="item.key"
            :class="{checked: checkedKeys.includes(item.key)}"
            @mouseenter="handleKeyMouseenter('userTags', item.key, $event)"
            @click="handleKeyClick(item.key)">
            <div class="title d-flex align-items-center">
              <div class="flex-fill mr-4 text-truncate">{{ getTagTitle(item.key) }}</div>
              <a-icon class="check-icon" type="check" />
            </div>
          </li>
        </ul>
        <ul class="tag-list" v-if="showExtTags">
          <li class="tag-tip">外部标签</li>
          <li
            class="tag-item"
            v-for="item of extTags"
            :key="item.key"
            :class="{checked: checkedKeys.includes(item.key)}"
            @mouseenter="handleKeyMouseenter('extTags', item.key, $event)"
            @click="handleKeyClick(item.key)">
            <div class="title d-flex align-items-center">
              <div class="flex-fill mr-4 text-truncate">{{ getTagTitle(item.key) }}</div>
              <a-icon class="check-icon" type="check" />
            </div>
          </li>
        </ul>
        <ul :style="valueWrapStyle" v-if="showValue" class="tag-list values-wrap">
          <li class="d-flex align-items-center tag-tip">
            <div class="flex-fill" style="font-size: 12px;">标签值：</div>
            <div class="val-search"><input style="width: 145px;" :value="search" placeholder="关键字搜索" @input="handleSearch" /></div>
          </li>
          <template v-if="currentValue.length <= 0">
            <li>未匹配到有效值</li>
          </template>
          <template v-else v-for="item of currentValue">
            <li
              v-if="item"
              class="tag-item"
              :key="item"
              :class="{ checked: value[mouseenterKey] && value[mouseenterKey].length > 0 && value[mouseenterKey].includes(item)}"
              @click="handleKeyClick(mouseenterKey, item)">
              <div class="title d-flex align-items-center">
                <div class="flex-fill mr-4">{{ item }}</div>
                <a-icon class="check-icon" type="check" />
              </div>
            </li>
          </template>
        </ul>
      </div>
    </template>
  </a-popover>
</template>

<script>
import * as R from 'ramda'
import { mapGetters } from 'vuex'
import debounce from 'lodash/debounce'
import Clickoutside from '@/directives/clickoutside'
import { getTagTitle } from '@/utils/common/tag'

export default {
  name: 'TagSelect',
  directives: {
    Clickoutside,
  },
  props: {
    value: {
      type: Object,
      required: true,
    },
    buttonText: {
      type: String,
      default: '标签',
    },
    params: Object,
    ignoreKeys: {
      type: Array,
      default: () => ([]),
    },
    filterWithoutUserMeta: Boolean,
    multiple: Boolean,
  },
  data () {
    return {
      visible: false,
      loading: false,
      userTags: [],
      extTags: [],
      valueWrapTop: 0,
      mouseenterKey: null,
      mouseenterType: null,
      search: '',
      withoutUserMetaKey: 'without_user_meta',
    }
  },
  computed: {
    ...mapGetters(['scope']),
    checkedKeys () {
      return Object.keys(this.value)
    },
    getParams () {
      let ret = {
        limit: 0,
        scope: this.scope,
        with_user_meta: true,
      }
      if (this.resource) ret.resources = this.resource
      if (this.params) {
        ret = Object.assign({}, ret, this.params)
      }
      return ret
    },
    currentTag () {
      let tags = []
      if (this.mouseenterType === 'userTags') {
        tags = this.userTags
      }
      if (this.mouseenterType === 'extTags') {
        tags = this.extTags
      }
      return tags
    },
    showUserTags () {
      return this.userTags.length > 0
    },
    showExtTags () {
      return this.extTags.length > 0
    },
    currentValue () {
      if (!this.mouseenterType || !this.mouseenterKey) return []
      let ret = []
      const obj = R.find(R.propEq('key', this.mouseenterKey))(this.currentTag)
      if (!this.search) {
        ret = obj.value
      } else {
        ret = obj.value.filter(val => val.includes(this.search))
      }
      return ret
    },
    showValue () {
      return this.currentValue && this.currentValue.some(val => val !== null)
    },
    valueWrapStyle () {
      return {
        top: `${this.valueWrapTop}px`,
      }
    },
  },
  destroyed () {
    this.manager = null
  },
  created () {
    this.manager = new this.$Manager('metadatas')
    this.debounceHandleSearchInput = debounce(this.handleSearchInput, 500)
  },
  methods: {
    handleVisibleChange (visible) {
      if (visible) {
        this.fetchTags()
      }
    },
    async fetchTags () {
      this.loading = true
      try {
        const response = await this.manager.get({ id: 'tag-value-pairs', params: this.getParams })
        const data = response.data.data || []
        this.genTags(data)
      } finally {
        this.loading = false
      }
    },
    genTags (data) {
      let userRet = []
      let extRet = []
      for (let i = 0, len = data.length; i < len; i++) {
        const item = data[i]
        const isUserKey = item.key.startsWith('user:')
        const isExtKey = item.key.startsWith('ext:')
        if (this.ignoreKeys.length > 0 && this.ignoreKeys.includes(item.key)) continue
        let temp
        if (isUserKey) {
          temp = userRet
        }
        if (isExtKey) {
          temp = extRet
        }
        const index = R.findIndex(R.propEq('key', item.key))(temp)
        if (index !== -1) {
          if (temp[index]['value'] && !temp[index]['value'].includes(item.value)) {
            temp[index]['value'].push(item.value)
          }
        } else {
          if (item.value) {
            temp.push({ key: item.key, value: [item.value] })
          } else {
            temp.push({ key: item.key, value: [] })
          }
        }
      }
      const sortByKeyCaseInsensitive = R.sortBy(R.compose(R.toLower, R.prop('key')))
      this.userTags = sortByKeyCaseInsensitive(userRet)
      this.extTags = sortByKeyCaseInsensitive(extRet)
    },
    getTagTitle,
    handleSearchInput (val) {
      this.search = val
    },
    handleSearch (e) {
      const val = e.target.value
      this.$nextTick(() => {
        this.debounceHandleSearchInput(val)
      })
    },
    handleKeyMouseenter (type, key, evt) {
      this.search = ''
      const tagWrapTop = this.$refs['tag-wrap'].getBoundingClientRect()['top']
      const targetTop = evt.target.getBoundingClientRect()['top']
      this.valueWrapTop = targetTop - tagWrapTop + 10
      this.mouseenterKey = key
      this.mouseenterType = type
    },
    handleKeyClick (key, val) {
      let newValue = { ...this.value }
      if (this.multiple) {
        if (val) {
          if (newValue[key]) {
            const index = R.indexOf(val, newValue[key])
            if (index !== -1) {
              newValue[key].splice(index, 1)
            } else {
              newValue[key].push(val)
            }
          } else {
            newValue[key] = [val]
          }
        } else {
          if (this.checkedKeys.includes(key)) {
            delete newValue[key]
          } else {
            newValue[key] = []
          }
        }
      } else {
        if (val) {
          if (newValue[key] === val) {
            delete newValue[key]
          } else {
            newValue[key] = [val]
          }
        } else {
          if (this.checkedKeys.includes(key)) {
            delete newValue[key]
          } else {
            newValue[key] = [val]
          }
        }
      }
      this.$emit('input', newValue)
      this.$emit('change', newValue)
    },
  },
}
</script>

<style lang="scss">
.tag-overlay .ant-popover-inner-content {
  padding: 0 !important;
}
</style>

<style lang="scss" scoped>
.tag-wrap {
  width: 180px;
  max-height: 400px;
  overflow: hidden;
  overflow-y: auto;
  > ul {
    list-style: none;
    margin: 0;
    padding: 0;
    > li {
      padding: 7px 16px 7px 16px;
    }
  }
}
.tag-tip {
  background-color: #F5F6FA;
}
.tag-item {
  cursor: pointer;
  background-color: #fff;
  white-space: pre-line;
  word-break: break-all;
  border-bottom: 1px solid #eee;
  position: relative;
  .check-icon {
    display: none;
    font-size: 12px;
  }
  &.checked {
    .check-icon {
      display: block;
    }
  }
  &:hover {
    background-color: #F9F9FA;
  }
}
.values-wrap {
  max-height: 400px;
  overflow: hidden;
  overflow-y: auto;
  background-color: #fff;
  position: absolute;
  left: 159px;
  left: -230px;
  top: 0;
  transition: all .3s ease;
  width: 230px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
}
</style>
