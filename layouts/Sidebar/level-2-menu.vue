<template>
  <div class="level-2-shade" v-show="visible">
    <div
      class="level-2-menu"
      v-if="menus">
      <div class="title">{{ label }}</div>
      <div
        class="level-3-item"
        v-for="(citem, cidx) of menus"
        :key="cidx">
        <div
          class="group-menu"
          v-if="citem.submenus">
          <div class="level-3-group-title">{{ getLabel(citem.meta) }}</div>
          <router-link
            v-for="(sitem, sidx) of citem.submenus"
            :key="sidx"
            class="menu-item"
            :to="sitem.path"
            tag="a"
            active-class="active">
            {{ getLabel(sitem.meta) }}
          </router-link>
        </div>
        <router-link
          v-else
          class="menu-item"
          :to="citem.path"
          tag="a"
          active-class="active">
          {{ getLabel(citem.meta) }}
        </router-link>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'Level2Menu',
  props: {
    currentMenu: {
      type: Object,
      required: true,
    },
    showMenu: {
      type: Function,
      required: true,
    },
    visible: {
      type: Boolean,
      default: false,
    },
  },
  computed: {
    label () {
      return this.currentMenu.meta.label
    },
    menus () {
      const menus = this.currentMenu.menus
      if (!menus) {
        return null
      }
      const res = []
      menus.forEach(m2item => {
        const m2 = { ...m2item }
        if (this.showMenu(m2)) {
          if (m2.submenus) {
            let flag = false
            const submenus = []
            m2.submenus.forEach(m3item => {
              if (this.showMenu(m3item)) {
                submenus.push(m3item)
                flag = true
              }
            })
            if (flag) {
              m2.submenus = submenus
              res.push(m2)
            }
          } else {
            res.push(m2)
          }
        }
      })
      return res
    },
  },
  methods: {
    getLabel (meta) {
      if (meta.i18n) {
        return this.$t(meta.label)
      }
      return meta.label
    },
  },
}
</script>

<style lang="scss" scoped>
.level-2-shade {
  position: absolute;
  left: 64px;
  width: 160px;
  top: 0;
  bottom: 0;
  background-color: rgb(247, 248, 250);
  box-shadow: 1px 0 6px 0 rgba(165,192,207,.3);
}

.level-2-menu {
  width: 100%;
  height: 100%;
  font-size: 14px;
  color: #fff;
  transition: left .2s;
  padding: 24px 0 0 23px;
  overflow-y: auto;
  .title {
    color: #435a71;
    font-size: 18px;
    margin-bottom: 16px;
  }
}

.level-3-item {
  .group-menu {
    margin-bottom: 10px;
  }
  .level-3-group-title {
    font-size: 12px;
    color: #53627C;
    line-height: 24px;
    margin-left: 3px;
    margin-bottom: 14px;
    margin-top: 14px;
  }
  .menu-item {
    display: block;
    padding-bottom: 4px;
    padding-top: 6px;
    padding-left: 20px;
    font-size: 14px;
    color: #43444B;
    position: relative;
    cursor: pointer;
    &::after {
      position: absolute;
      content: '';
      left: 0;
      top: 0;
      height: 100%;
      width: 1px;
      background-color: #D8D8D8;
      transition: width .2s ease;
    }
    &:hover, &.active {
      text-decoration: none;
      &::after {
        width: 4px;
        background-color: #1890ff;
      }
    }
  }
  > .menu-item {
    margin-bottom: 25px;
  }
  & + & {
    > .menu-item {
      margin-top: 25px;
    }
  }
}
</style>
