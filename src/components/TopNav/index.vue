<template>
  <el-menu
    :default-active="activeMenu"
    mode="horizontal"
    @select="handleSelect"
    :ellipsis="false"
  >
    <template v-for="(item, index) in topMenus">
      <el-menu-item :style="{'--theme': theme}" :index="item.path" :key="index" v-if="index < visibleNumber">

      <div style="display: flex; flex-direction: column; align-items: center;">
        <svg-icon
          v-if="item.meta && item.meta.icon && item.meta.icon !== '#'"
          :icon-class="item.meta.icon"
          style="margin: 0;"/>
        <span class="menu-title">{{ item.meta.title }}</span>
      </div>

      </el-menu-item>
    </template>

    <!-- 顶部菜单超出数量折叠 -->
    <el-sub-menu :style="{'--theme': theme}" index="more" v-if="topMenus.length > visibleNumber">
      <template #title>更多菜单</template>
      <template v-for="(item, index) in topMenus">
        <el-menu-item
          :index="item.path"
          :key="index"
          v-if="index >= visibleNumber">

          <div style="display: flex; flex-direction: column; align-items: center;">
            <svg-icon
              v-if="item.meta && item.meta.icon && item.meta.icon !== '#'"
              :icon-class="item.meta.icon"
              style="margin: 0;"/>
            <span>{{ item.meta.title }}</span>
          </div>

        </el-menu-item>
      </template>
    </el-sub-menu>
  </el-menu>
</template>

<script setup>
import { constantRoutes } from "@/router"
import { isHttp } from '@/utils/validate'
import useAppStore from '@/store/modules/app'
import useSettingsStore from '@/store/modules/settings'
import usePermissionStore from '@/store/modules/permission'
import variables from '@/assets/styles/variables.module.scss'

// 顶部栏初始数
const visibleNumber = ref(null)
// 当前激活菜单的 index
const currentIndex = ref(null)
// 隐藏侧边栏路由
const hideList = ['/index', '/user/profile']

const appStore = useAppStore()
const settingsStore = useSettingsStore()
const permissionStore = usePermissionStore()
const route = useRoute()
const router = useRouter()

// 主题颜色
const theme = computed(() => settingsStore.theme)
const sideTheme = computed(() => settingsStore.sideTheme)
// 所有的路由信息
const routers = computed(() => permissionStore.topbarRouters)

// 顶部显示菜单
const topMenus = computed(() => {
  let topMenus = []
  routers.value.map((menu) => {
    if (menu.hidden !== true) {
      // 兼容顶部栏一级菜单内部跳转
      if (menu.path === '/' && menu.children) {
          topMenus.push(menu.children[0])
      } else {
          topMenus.push(menu)
      }
    }
  })
  return topMenus
})


// 获取菜单背景色
const getMenuBackground = computed(() => {
  if (settingsStore.isDark) {
    return 'var(--sidebar-bg)'
  }
  return sideTheme.value === 'theme-dark' ? variables.menuBg : variables.menuLightBg
})

// 获取菜单文字颜色
const getMenuTextColor = computed(() => {
  if (settingsStore.isDark) {
    return 'var(--sidebar-text)'
  }
  return sideTheme.value === 'theme-dark' ? variables.menuText : variables.menuLightText
})


// 设置子路由
const childrenMenus = computed(() => {
  let childrenMenus = []
  routers.value.map((router) => {
    for (let item in router.children) {
      if (router.children[item].parentPath === undefined) {
        if(router.path === "/") {
          router.children[item].path = "/" + router.children[item].path
        } else {
          if(!isHttp(router.children[item].path)) {
            router.children[item].path = router.path + "/" + router.children[item].path
          }
        }
        router.children[item].parentPath = router.path
      }
      childrenMenus.push(router.children[item])
    }
  })
  return constantRoutes.concat(childrenMenus)
})

// 默认激活的菜单
const activeMenu = computed(() => {
  const path = route.path
  let activePath = path
  if (path !== undefined && path.lastIndexOf("/") > 0 && hideList.indexOf(path) === -1) {
    const tmpPath = path.substring(1, path.length)
    if (!route.meta.link) {
      activePath = "/" + tmpPath.substring(0, tmpPath.indexOf("/"))
      appStore.toggleSideBarHide(false)
    }
  } else if(!route.children) {
    activePath = path
    appStore.toggleSideBarHide(true)
  }
  activeRoutes(activePath)
  return activePath
})

function setVisibleNumber() {
  const width = document.body.getBoundingClientRect().width / 3
  visibleNumber.value = parseInt(width / 85)
}

function handleSelect(key, keyPath) {
  currentIndex.value = key
  const route = routers.value.find(item => item.path === key)
  if (isHttp(key)) {
    // http(s):// 路径新窗口打开
    window.open(key, "_blank")
  } else if (!route || !route.children) {
    // 没有子路由路径内部打开
    const routeMenu = childrenMenus.value.find(item => item.path === key)
    if (routeMenu && routeMenu.query) {
      let query = JSON.parse(routeMenu.query)
      router.push({ path: key, query: query })
    } else {
      router.push({ path: key })
    }
    appStore.toggleSideBarHide(true)
  } else {
    // 显示左侧联动菜单
    activeRoutes(key)
    appStore.toggleSideBarHide(false)
  }
}

function activeRoutes(key) {
  let routes = []
  if (childrenMenus.value && childrenMenus.value.length > 0) {
    childrenMenus.value.map((item) => {
      if (key == item.parentPath || (key == "index" && "" == item.path)) {
        routes.push(item)
      }
    })
  }
  if(routes.length > 0) {
    permissionStore.setSidebarRouters(routes)
  } else {
    appStore.toggleSideBarHide(true)
  }
  return routes
}

onMounted(() => {
  window.addEventListener('resize', setVisibleNumber)
})

onBeforeUnmount(() => {
  window.removeEventListener('resize', setVisibleNumber)
})

onMounted(() => {
  setVisibleNumber()
})

</script>

<style lang="scss">
.topmenu-container.el-menu--horizontal > .el-menu-item {
  float: left;
  height: 70px !important;
  line-height: 70px !important;
  // 顶部菜单文字字体颜色
  color: var(--847c2ff5-getMenuTextColor);
  // color: #999093 !important;
  padding: 0 5px !important;
  margin: 0 10px !important;
}

.topmenu-container.el-menu--horizontal > .el-menu-item.is-active, .el-menu--horizontal > .el-sub-menu.is-active .el-submenu__title {
  border-bottom: 2px solid #{'var(--theme)'} !important;
  color: #303133;
}

/* sub-menu item */
.topmenu-container.el-menu--horizontal > .el-sub-menu .el-sub-menu__title {
  float: left;
  height: 50px !important;
  line-height: 50px !important;
  color: #999093 !important;
  padding: 0 5px !important;
  margin: 0 10px !important;
}

/* 背景色隐藏 */
.topmenu-container.el-menu--horizontal>.el-menu-item:not(.is-disabled):focus, .topmenu-container.el-menu--horizontal>.el-menu-item:not(.is-disabled):hover, .topmenu-container.el-menu--horizontal>.el-submenu .el-submenu__title:hover {
  background-color: #ffffff;
}

/* 图标右间距 */
.topmenu-container .svg-icon {
  margin-right: 4px;
}

/* topmenu more arrow */
.topmenu-container .el-sub-menu .el-sub-menu__icon-arrow {
  position: static;
  vertical-align: middle;
  margin-left: 8px;
  margin-top: 0px;
}

.topmenu-container .el-menu-item,
.topmenu-container .el-sub-menu .el-menu-item {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  height: 50px !important;
  line-height: normal !important;
}

.topmenu-container .el-menu-item .svg-icon {
  margin-right: 0 !important;
}


// 取消水平模式菜单下的下划线边框
.el-menu--horizontal {
  border-bottom: none !important;

  background-color: #1b5fb5;
}

//顶部菜单栏
.el-menu-item{
  color: v-bind(getMenuTextColor);

  //菜单选中背景颜色
  &.is-active {
    color: var(--menu-active-text, #409eff);
    background-color: var(--menu-hover, #1b5fb5) !important;
  }
  //顶部菜单悬浮背景颜色
  &:hover {
    background-color: var(--menu-hover, #1b5fb5) !important;
  }
}
//顶部菜单图标
.topmenu-container .el-menu-item .svg-icon {
  transform: scale(2); // 放大图标
  margin-top: 30px; // 向下移动图标，靠近字体
}
//顶部菜单文字
.topmenu-container .el-menu-item .menu-title {
  height: 20px;
  transform: translateY(-10px); // 文字整体往上移动4px

}
</style>
