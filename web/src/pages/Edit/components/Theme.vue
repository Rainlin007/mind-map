<template>
  <Sidebar ref="sidebar" :title="$t('theme.title')">
    <div class="themeGroupList" :class="{ isDark: isDark }">
      <el-tabs v-model="activeName" class="tabBox">
        <el-tab-pane
          v-for="group in groupList"
          :key="group.name"
          :label="group.name"
          :name="group.name"
        ></el-tab-pane>
      </el-tabs>
      <div class="themeListTheme customScrollbar">
        <div
          class="themeItem"
          v-for="item in currentList"
          :key="item.value"
          @click="useTheme(item)"
          :class="{ active: item.value === theme }"
        >
          <div class="imgBox">
            <img :src="item.img || themeImgMap[item.value]" alt="" />
          </div>
          <div class="name">{{ item.name }}</div>
        </div>
        <!-- 背景花纹 -->
        <div class="patternSection">
          <div class="patternTitle">{{ $t('theme.bgPattern') }}</div>
          <div class="patternGrid">
            <div
              class="patternCard"
              v-for="p in patternList"
              :key="p.value"
              :class="{ active: backgroundPattern === p.value }"
              @click="selectPattern(p.value)"
            >
              <div class="patternPreview" :style="getPatternPreviewStyle(p.value)"></div>
              <div class="patternName">{{ p.name }}</div>
            </div>
          </div>
          <div class="patternColorRow" v-if="backgroundPattern && backgroundPattern !== 'none'">
            <span class="patternColorLabel">{{ $t('theme.patternColor') }}</span>
            <input
              type="color"
              class="patternColorPicker"
              :value="bgPatternColor"
              @input="onPatternColorChange($event.target.value)"
            />
            <input
              type="range"
              class="patternOpacitySlider"
              min="5"
              max="100"
              step="5"
              :value="bgPatternOpacity"
              @input="onPatternOpacityChange($event.target.value)"
            />
            <span class="patternOpacityValue">{{ bgPatternOpacity }}%</span>
          </div>
        </div>
      </div>
    </div>
  </Sidebar>
</template>

<script>
import Sidebar from './Sidebar.vue'
import { storeData } from '@/api'
import { mapState, mapMutations } from 'vuex'
import themeImgMap from 'simple-mind-map-plugin-themes/themeImgMap'
import themeList from 'simple-mind-map-plugin-themes/themeList'

// 主题
export default {
  components: {
    Sidebar
  },
  props: {
    data: {
      type: [Object, null],
      default: null
    },
    mindMap: {
      type: Object
    }
  },
  data() {
    return {
      themeList: [
        {
          name: '默认主题',
          value: 'default',
          dark: false
        },
        ...themeList
      ].reverse(),
      themeImgMap,
      theme: '',
      activeName: '',
      defaultGroupList: []
    }
  },
  computed: {
    ...mapState({
      isDark: state => state.localConfig.isDark,
      activeSidebar: state => state.activeSidebar,
      extendThemeGroupList: state => state.extendThemeGroupList,
      backgroundPattern: state => state.localConfig.backgroundPattern,
      bgPatternColor: state => state.localConfig.bgPatternColor,
      bgPatternOpacity: state => state.localConfig.bgPatternOpacity
    }),

    groupList() {
      return [...this.defaultGroupList, ...this.extendThemeGroupList]
    },

    currentList() {
      return this.groupList.find(item => {
        return item.name === this.activeName
      }).list
    },

    patternList() {
      return [
        { value: 'none', name: this.$t('theme.patternNone') },
        { value: 'dots', name: this.$t('theme.patternDots') },
        { value: 'grid', name: this.$t('theme.patternGrid') },
        { value: 'crossDot', name: this.$t('theme.patternCrossDot') }
      ]
    }
  },
  watch: {
    activeSidebar(val) {
      if (val === 'theme') {
        this.theme = this.mindMap.getTheme()
        this.$refs.sidebar.show = true
      } else {
        this.$refs.sidebar.show = false
      }
    }
  },
  created() {
    this.initGroup()
    this.theme = this.mindMap.getTheme()
    this.mindMap.on('view_theme_change', this.handleViewThemeChange)
  },
  beforeDestroy() {
    this.mindMap.off('view_theme_change', this.handleViewThemeChange)
  },
  methods: {
    ...mapMutations(['setLocalConfig']),

    handleViewThemeChange() {
      this.theme = this.mindMap.getTheme()
      this.handleDark()
    },

    initGroup() {
      const baiduThemes = [
        'default',
        'skyGreen',
        'classic2',
        'classic3',
        'classicGreen',
        'classicBlue',
        'blueSky',
        'brainImpairedPink',
        'earthYellow',
        'freshGreen',
        'freshRed',
        'romanticPurple',
        'pinkGrape',
        'mint'
      ]
      const baiduList = []
      const classicsList = []
      this.themeList.forEach(item => {
        if (baiduThemes.includes(item.value)) {
          baiduList.push(item)
        } else if (!item.dark) {
          classicsList.push(item)
        }
      })
      this.defaultGroupList = [
        {
          name: this.$t('theme.classics'),
          list: classicsList
        },
        {
          name: this.$t('theme.dark'),
          list: this.themeList.filter(item => {
            return item.dark
          })
        },
        {
          name: this.$t('theme.simple'),
          list: baiduList
        }
      ]
      this.activeName = this.defaultGroupList[0].name
    },

    useTheme(theme) {
      if (theme.value === this.theme) return
      this.theme = theme.value
      this.handleDark()
      const customThemeConfig = this.mindMap.getCustomThemeConfig()
      const hasCustomThemeConfig = Object.keys(customThemeConfig).length > 0
      if (hasCustomThemeConfig) {
        this.$confirm(this.$t('theme.coverTip'), this.$t('theme.tip'), {
          confirmButtonText: this.$t('theme.cover'),
          cancelButtonText: this.$t('theme.reserve'),
          type: 'warning',
          distinguishCancelAndClose: true,
          callback: action => {
            if (action === 'confirm') {
              this.mindMap.setThemeConfig({}, true)
              this.data.theme.config = {}
              this.changeTheme(theme, {})
            } else if (action === 'cancel') {
              this.changeTheme(theme, customThemeConfig)
            }
          }
        })
      } else {
        this.changeTheme(theme, customThemeConfig)
      }
    },

    changeTheme(theme, config) {
      this.$bus.$emit('showLoading')
      this.mindMap.setTheme(theme.value)
      storeData({
        theme: {
          template: theme.value,
          config
        }
      })
    },

    handleDark() {
      const extendThemeList = []
      this.extendThemeGroupList.forEach(group => {
        extendThemeList.push(...group.list)
      })
      let target = [...this.themeList, ...extendThemeList].find(item => {
        return item.value === this.theme
      })
      this.setLocalConfig({
        isDark: target.dark
      })
    },

    selectPattern(value) {
      this.setLocalConfig({ backgroundPattern: value })
    },

    onPatternColorChange(value) {
      this.setLocalConfig({ bgPatternColor: value })
    },

    onPatternOpacityChange(value) {
      this.setLocalConfig({ bgPatternOpacity: parseInt(value) })
    },

    getPatternPreviewStyle(value) {
      const dark = this.isDark
      const bg = dark ? '#2c2c2c' : '#f5f5f5'
      const c = dark ? 'rgba(255,255,255,0.15)' : 'rgba(0,0,0,0.10)'
      const cs = dark ? 'rgba(255,255,255,0.20)' : 'rgba(0,0,0,0.15)'
      const patterns = {
        none: { backgroundColor: bg },
        dots: {
          backgroundColor: bg,
          backgroundImage: `radial-gradient(circle, ${cs} 1px, transparent 1px)`,
          backgroundSize: '20px 20px'
        },
        grid: {
          backgroundColor: bg,
          backgroundImage: `linear-gradient(${c} 1px, transparent 1px), linear-gradient(90deg, ${c} 1px, transparent 1px)`,
          backgroundSize: '24px 24px'
        },
        linesHorizontal: {
          backgroundColor: bg,
          backgroundImage: `linear-gradient(${c} 1px, transparent 1px)`,
          backgroundSize: '10px 16px'
        },
        diagonal: {
          backgroundColor: bg,
          backgroundImage: `repeating-linear-gradient(45deg, ${c}, ${c} 1px, transparent 1px, transparent 10px)`
        },
        crossDot: {
          backgroundColor: bg,
          backgroundImage: `linear-gradient(${cs} 1px, transparent 1px), linear-gradient(90deg, ${cs} 1px, transparent 1px)`,
          backgroundSize: '20px 20px'
        }
      }
      return patterns[value] || patterns.none
    }
  }
}
</script>

<style lang="less" scoped>
.themeGroupList {
  display: flex;
  flex-direction: column;
  overflow: hidden;
  height: 100%;

  &.isDark {
    .name {
      color: #fff;
    }
  }

  .tabBox {
    flex-shrink: 0;

    /deep/ .el-tabs__nav-wrap {
      display: flex;
      justify-content: center;
    }
  }

  .themeListTheme {
    height: 100%;
    overflow-y: auto;
    padding: 0 20px;

    .themeItem {
      width: 100%;
      cursor: pointer;
      border-bottom: 1px solid #e9e9e9;
      margin-bottom: 20px;
      padding-bottom: 20px;
      transition: all 0.2s;
      border: 3px solid transparent;
      border-radius: 5px;
      overflow: hidden;

      &:last-of-type {
        border: none;
      }

      &:hover {
        box-shadow: 0 1px 2px -2px rgba(0, 0, 0, 0.16),
          0 3px 6px 0 rgba(0, 0, 0, 0.12), 0 5px 12px 4px rgba(0, 0, 0, 0.09);
      }

      &.active {
        border: 3px solid rgb(154, 198, 250);
      }

      .imgBox {
        width: 100%;

        img {
          width: 100%;
        }
      }
      .name {
        text-align: center;
        font-size: 14px;
      }
    }

    .patternSection {
      padding-top: 10px;
      border-top: 1px solid #e9e9e9;
      margin-top: 10px;

      .patternTitle {
        font-size: 14px;
        font-weight: 500;
        margin-bottom: 12px;
        color: rgba(26, 26, 26, 0.9);
      }

      .patternGrid {
        display: grid;
        grid-template-columns: repeat(3, 1fr);
        gap: 10px;
      }

      .patternCard {
        cursor: pointer;
        border: 2px solid transparent;
        border-radius: 6px;
        overflow: hidden;
        transition: all 0.2s;

        &:hover {
          border-color: #c0d8f0;
        }

        &.active {
          border-color: rgb(154, 198, 250);
        }

        .patternPreview {
          width: 100%;
          height: 48px;
          border-radius: 4px 4px 0 0;
        }

        .patternName {
          text-align: center;
          font-size: 12px;
          padding: 4px 0;
          color: rgba(26, 26, 26, 0.7);
        }
      }

      .patternColorRow {
        display: flex;
        align-items: center;
        gap: 8px;
        margin-top: 12px;

        .patternColorLabel {
          font-size: 12px;
          flex-shrink: 0;
          color: rgba(26, 26, 26, 0.7);
        }

        .patternColorPicker {
          width: 28px;
          height: 24px;
          border: 1px solid #ddd;
          border-radius: 3px;
          padding: 0;
          cursor: pointer;
          background: none;
          flex-shrink: 0;
        }

        .patternOpacitySlider {
          flex: 1;
          min-width: 48px;
        }

        .patternOpacityValue {
          font-size: 11px;
          min-width: 30px;
          text-align: right;
          color: rgba(26, 26, 26, 0.5);
          flex-shrink: 0;
        }
      }
    }
  }
}

.isDark {
  .patternSection {
    border-top-color: hsla(0, 0%, 100%, 0.1);

    .patternTitle {
      color: hsla(0, 0%, 100%, 0.9);
    }

    .patternCard {
      .patternName {
        color: hsla(0, 0%, 100%, 0.6);
      }
    }

    .patternColorRow {
      .patternColorLabel {
        color: hsla(0, 0%, 100%, 0.6);
      }

      .patternColorPicker {
        border-color: hsla(0, 0%, 100%, 0.2);
      }

      .patternOpacityValue {
        color: hsla(0, 0%, 100%, 0.5);
      }
    }
  }
}
</style>
