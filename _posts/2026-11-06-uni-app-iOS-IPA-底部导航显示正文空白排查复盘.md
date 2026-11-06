---
layout:     post
title:      uni-app iOS IPA 只有底部导航、组件和图标不显示
subtitle:   从编译器错配到 easycom 注册、图标资源和最终产物核验
date:       2026-11-06
author:     Trojan
header-img: img/post-bg-58.jpg
catalog: true
tags:
    - uni-app
    - iOS
    - HBuilderX
    - Vue
    - Vite
    - Webpack
    - Debug
    - 工程复盘
---

# uni-app iOS IPA 只有底部导航、组件和图标不显示：一次 HBuilderX 正式打包排查复盘

最近在悦途骑行客户端遇到一个很容易把人带偏的问题：同一份 uni-app 代码，用 HBuilderX“运行到 iOS App 基座”时完全正常；云打包成 IPA、安装到 iPhone 后，却只剩底部导航栏，首页和其他页面的正文都不见了。

最开始我也沿着常见方向排查：是不是正式包没有 Mock 数据？是不是 iOS CSS 兼容性？是不是证书类型不对？是不是打包日志里的 warning 导致页面没有渲染？这些猜测都合理，但都没有击中根因。

最后真正解决问题的不是改页面，而是把 `manifest.json` 里的 `vueVersion` 从数字 `3` 改成字符串 `"3"`。这个看起来很小的类型差异，让 HBuilderX 5.34 选错了 uni-app 编译器：Vue 3/Vite 项目被云端按 Vue 2/Webpack 链路编译，最终 IPA 中的页面 bundle 与项目 runtime 不匹配。

这篇文章记录完整的排查过程，也记录哪些现象可以帮助我们更快定位同类问题。

## 一、现象：Debug 正常，IPA 只剩导航

当时的对比很明确：

| 运行方式 | 结果 |
| --- | --- |
| HBuilderX 运行到 iOS App 基座 | 首页正文、页面内容、底部导航都正常 |
| 云打包 IPA 安装到手机 | 底部导航显示，正文区域没有内容 |

切换底部导航后，其他页面也一样没有正文。这一点很关键：如果只有首页接口失败，应该优先查首页数据；如果每个页面都没有内容，问题更像是公共运行时或资源加载失败。

它也不是完全白屏。底部导航仍然显示，说明 App 原生壳和部分 manifest 配置已经被加载。可以把它理解成三层：

1. 原生层：App 是否能启动，底部导航是否出现；
2. WebView 层：`app-service.js` 和页面 bundle 是否能执行；
3. 业务层：Mock、HTTP API、图片和其他数据是否返回。

这次原生层看起来正常，业务层还没轮到排查，真正需要先确认的是 WebView 层。

## 二、第一轮猜测：是不是 Mock 数据没有打进去

因为 Debug 基座中使用 Mock 数据，正式包又是另一条打包流程，所以第一个怀疑是：

> Debug 能显示，是因为基座能拿到 Mock；云打包后没有 Mock，接口返回空，所以正文没有了。

这个假设需要验证，但它有一个不太合理的地方：即使数据请求失败，页面通常仍会有页面标题、骨架屏、空状态或错误提示。所有页面连公共布局都不显示，更像是页面 JavaScript 没有成功启动。

后来项目统一调整为所有运行环境默认使用内置 Mock 数据，HTTP Mock 服务只保留给独立的网络边界测试。这样可以消除环境切换带来的变量，但这并不是最终修复。真正的证据来自打包产物。

## 三、日志显示“编译成功”，为什么还不能放心

云打包日志里有类似下面的内容：

```text
项目 client 编译成功。
项目 client 打包成功。
类型: iOS Appstore
使用开发证书，只能安装到测试设备
```

同时还有一条 PostCSS warning：

```text
start value has mixed support, consider using flex-start instead
```

这些信息很容易让人继续在错误方向上消耗时间。

“编译成功”只能说明打包流程没有中断，不能证明：

- 选中了正确的 Vue 编译器；
- 打进 IPA 的资源来自当前源码；
- WebView runtime 能执行这些资源。

开发证书提示影响签名和安装范围，与正文是否渲染没有直接关系。`start` 的 warning 是 CSS 兼容性建议，也不足以解释所有页面一起消失。

所以，下一步不再盯日志表面，而是检查 IPA 里面究竟有什么代码。

## 四、关键转折：直接检查 IPA 的 `app-service.js`

先从旧 IPA 中提取 WebView 的主 bundle：

```bash
unzip -p path/to/app.ipa 'Payload/*/www/app-service.js' > /tmp/app-service.js
head -c 180 /tmp/app-service.js
rg -o 'webpackJsonp|\\$createElement|_self\\._c|\\$set|createSSRApp|__vite__' \
  /tmp/app-service.js | sort | uniq -c
```

旧包的文件开头是：

```text
(this["webpackJsonp"]=this["webpackJsonp"]||[]).push([[
```

并且里面大量出现：

```text
$createElement
_self._c
$set
webpackJsonp
```

这不是当前项目应该生成的资源。当前项目的源码入口使用 `createSSRApp`，构建配置使用 `@dcloudio/vite-plugin-uni`，依赖也是 Vue 3。旧包却明显是 Vue 2/Webpack 产物。

这一步把问题从“页面为什么没数据”变成了一个更具体的问题：**为什么一个 Vue 3/Vite 项目会被编译成 Vue 2/Webpack bundle？**

## 五、确认项目和 HBuilderX 同时存在两条编译链

HBuilderX 5.34 的安装目录里同时有两套 uni-app 编译器：

```text
plugins/uniapp-cli       # Vue 2 / Webpack
plugins/uniapp-cli-vite  # Vue 3 / Vite
```

这解释了为什么“运行到 App 基座”和“云打包”可能表现不同：两条流程不一定使用同一个项目识别结果和缓存。

继续查看 HBuilderX 的项目识别规则，可以看到 Vue 3 的匹配条件是字符串：

```json
{"vueVersion":"3"}
```

而项目原来的 `manifest.json` 写成了：

```json
"vueVersion": 3
```

在普通 JSON 处理里，这只是数字和字符串的类型差异；在这版 HBuilderX 的识别逻辑里，它直接影响编译器选择。因为没有命中 Vue 3/Vite 规则，云打包走到了旧的 Vue 2/Webpack 链路。

## 六、为什么会出现“底部导航还在”

这个现象现在就能解释了：

1. `manifest.json` 中的 tabBar 配置被原生壳读取，所以底部导航还能显示；
2. 页面正文依赖 WebView 中的 JavaScript bundle；
3. bundle 是按 Vue 2/Webpack 生成的，而项目实际运行环境和源码是 Vue 3/Vite；
4. 页面入口没有按预期完成初始化，因此每个页面都像“没有内容”。

如果只看到底部导航，不能得出“页面组件全部写坏了”的结论。它更像是原生 manifest 已经生效，但 WebView 业务资源没有正常执行。

## 七、正确修复：修正 `vueVersion` 的类型

最终修改非常小：

```diff
- "vueVersion": 3
+ "vueVersion": "3"
```

同时把这条约束写入项目的 HBuilderX 运行说明，避免以后有人格式化 manifest 或手动改配置时又变回数字。

修改后，用 HBuilderX CLI 重新导出 App 资源：

```bash
/Applications/HBuilderX.app/Contents/MacOS/cli publish app \
  --platform APP \
  --type appResource \
  --project /path/to/apps/client
```

正确日志应明确出现：

```text
编译器版本：5.34（vue3）
项目 client 编译成功。
项目 client 导出成功，路径为：.../unpackage/resources
```

不能只看最后一行“导出成功”。如果日志仍然出现：

```text
plugins/uniapp-cli/bin/uniapp-cli.js
```

就说明仍在走旧编译器，应先处理项目识别或缓存。

## 八、修复后的产物验证

重新导出的资源位于 `unpackage/resources`，其中的 `app-service.js` 约 600 KB。检查旧产物特征：

```bash
rg -o 'webpackJsonp|\\$createElement|_self\\._c|\\$set' \
  unpackage/resources/__UNI__*/www/app-service.js
```

这次没有再匹配到 Vue 2/Webpack 的特征。文件开头也不再是 `webpackJsonp`，而是 Vite/esbuild 风格的 bundle。

同时完成了：

```bash
pnpm --dir apps/client typecheck
```

项目测试 71/71 通过，`git diff --check` 通过。

这些结果说明源码、类型检查、HBuilderX CLI 选择和本地导出资源已经对齐。

## 九、HBuilderX 仍然缓存旧结果怎么办

如果已经修改 `manifest.json`，日志却仍显示旧编译器，不要继续重复云打包。按下面的顺序处理：

1. 完全退出 HBuilderX；
2. 重新打开 HBuilderX；
3. 移除项目管理中旧的项目记录；
4. 重新导入完整的 `apps/client` 目录；
5. 再检查 `manifest.json` 确认是字符串 `"3"`；
6. 先用 CLI 或“发行 → 原生 App-云打包”确认日志写着 `（vue3）`；
7. 确认无 `Module parse failed`、`Unexpected token` 或 `ERROR Build failed` 后，再安装新 IPA。

正式包不要勾选“使用自定义调试基座”。自定义基座用于原生插件调试，不能作为正式 IPA 的资源来源。

## 十、哪些提示与本次根因无关

### 1. Mock 数据

Mock 只是数据来源。Mock 失败一般会表现为加载失败、空状态或请求错误，不会让所有页面的 JavaScript bundle 都无法执行。正确做法是先验证资源和 runtime，再查数据。

### 2. Apple Development 证书

开发证书决定 IPA 能安装到哪些测试设备，不决定 Vue 页面 bundle 是否执行。它需要在发布阶段处理，但不是这次正文不显示的原因。

### 3. PostCSS warning

`start value has mixed support` 建议改成 `flex-start`，可以单独修复以减少 warning，但它不能解释底部导航以外的全部页面都消失。

### 4. 自定义调试基座提示

自定义基座和正式云打包的用途不同。基座运行正常，只能说明某条调试链路正常，不能证明云打包选了相同的编译器。

## 十一、以后遇到“Debug 正常、正式包异常”的检查清单

我现在会按下面的顺序排查：

1. **先分层**：原生壳、WebView/JS、业务数据分别验证；
2. **查编译器**：确认 Vue 版本、Vite/Webpack 链路和实际日志路径；
3. **查产物**：解压 IPA，检查 `app-service.js` 是否符合项目技术栈；
4. **查入口配置**：核对 `manifest.json`、`main.ts`、`vite.config.ts` 和 `package.json`；
5. **查 WebView 错误**：用 Safari Web Inspector 或设备日志看 JS 异常；
6. **查 Mock/API**：确认数据请求、接口地址、权限和网络；
7. **最后查样式**：再看高度、安全区、图片、字体和布局；
8. **只验证新包**：记录 IPA 生成时间，不要把旧 IPA 的表现当成修复结果。

这个顺序的核心是“先产物，后数据；先编译器，后页面”。它能避免在错误的层级里反复修改和重复打包。

## 十二、最后的验证边界

这次已经完成了源码配置修复、类型检查、CLI 编译器验证和本地 App 资源检查。新的资源已经确认不再是旧 Vue 2/Webpack bundle。

## 十三、第二个根因：正式导出没有把 easycom 组件带进来

修复 Vue 版本后，页面正文可以继续执行，但正式包又出现了另一种很像“资源没显示”的现象：底部 PNG 导航还在，页面里的图标、搜索框、列表、标签页和按钮全部消失。

这次用 HBuilderX 5.34 自带 CLI 导出 `appResource`，直接检查 `unpackage/resources/__UNI__*/www/`，发现 `app-service.js` 里保留了：

```js
resolveComponent("y-icon")
resolveComponent("t-search")
```

但组件定义没有进入正式 bundle，`app.css` 里也找不到 `.t-search__...` 这类 TDesign 组件样式。调试基座和 Vite CLI 构建正常，是因为它们能根据 `pages.json` 的 `easycom.custom` 自动解析组件；HBuilderX 5.34 的这条正式导出链路没有可靠完成这一步。

这解释了当时的画面：PNG、纯文本和手写的 `+` 不依赖 Vue 组件注册，因此还在；`YetuIcon`、搜索框和 TDesign 组件依赖运行时组件定义，因此全部变成空白或没有节点。此时继续改 Mock 数据、图片 URL 或 CSS 都不会解决问题。

### 用全局注册给正式包一个确定入口

项目新增 `apps/client/shared/ui/global-components.ts`，显式导入并登记 `y-icon` 和 TDesign 组件：

```ts
const GLOBAL_COMPONENTS = {
  'y-icon': YetuIcon,
  't-search': TSearch,
  't-button': TButton,
  // 其余 t-avatar、t-cell、t-tabs、t-tag 等组件
}

export function registerGlobalComponents(app: App) {
  for (const [name, component] of Object.entries(GLOBAL_COMPONENTS)) {
    app.component(name, component)
  }
}
```

`main.ts` 在创建应用时调用它：

```ts
export function createApp() {
  const app = createSSRApp(App)
  app.use(createPinia())
  registerGlobalComponents(app)
  return { app }
}
```

`pages.json` 的 easycom 仍然保留，因为它对开发体验有用；但它不再是正式 App 能否找到组件的唯一机制。以后在 `vendor/tdesign-uniapp` 中增加新组件时，必须同时加入全局注册表。

### 导出后检查定义和样式，而不是只看调用点

```bash
WWW=apps/client/unpackage/resources/__UNI__3039980/www
SERVICE="$WWW/app-service.js"
CSS="$WWW/app.css"

rg -n 'const .*\{"y-icon"|__name:"YetuIcon"' "$SERVICE"
rg -n 'resolveComponent\("y-icon"\)|resolveComponent\("t-search"\)' "$SERVICE"
rg -n '\.t-search__label|\.t-search__input-box' "$CSS"
```

`resolveComponent` 仍然存在并不代表组件丢失，它只是模板调用点。必须同时看到组件定义和样式，才说明正式资源可以渲染。修复后的导出结果中可以看到全局组件表、`YetuIcon` 定义和 `.t-search__...` 样式。

## 十四、第三个根因：iOS 图标资源不能只配置一张带透明圆角的 PNG

桌面图标的问题又是独立的一层。旧配置只写了：

```json
{
  "ios": {
    "appstore": "static/app-icon.png"
  }
}
```

旧 PNG 是带透明通道的圆角图。iOS 会自己裁切圆角，源图应该是四角满铺、无透明通道的正方形；只给一张 `appstore` 图也无法覆盖 iPhone 和 iPad 的其它图标槽位。

现在由 `static/app-icon.svg` 维护源图，并生成 `static/app-icons/ios/` 下的 18 张 RGB PNG：

- App Store 1024×1024；
- iPhone App、通知、设置、Spotlight 的 2x/3x；
- iPad App、Pro App、通知、设置、Spotlight 的 1x/2x。

`manifest.json` 的 `app-plus.distribute.icons.ios` 已逐项指向这些文件。检查透明通道：

```bash
find apps/client/static/app-icons/ios -name '*.png' | wc -l  # 18
file apps/client/static/app-icons/ios/*.png
```

安装后若图标仍像旧图，先删除旧 App，再安装新 IPA。iOS SpringBoard 会缓存相同 Bundle ID 的图标；同时要确认检查的是新 IPA，而不是源码目录或上一次下载链接。

## 十五、把最终 IPA 当成唯一证据

HBuilderX 窗口显示“编译成功”和“打包成功”只说明流程完成，不说明所有组件、字体、媒体和图标已经进入最终包。当前建议固定检查三层：

1. **源码层**：`manifest.json` 的 `vueVersion` 是字符串，`main.ts` 调用了全局注册，图标 manifest 指向完整资源集；
2. **导出层**：`unpackage/resources` 中同时有组件定义、`.t-search__...` 样式、TDesign TTF、本地媒体和 18 张 iOS 图标；
3. **IPA 层**：`Assets.car` 含 `AppIcon` 1024 marketing rendition，`Info.plist` 含 `CFBundleIconName = AppIcon`，WebView bundle 能搜索到 `YetuIcon` 和 TDesign 组件定义。

IPA 可以这样检查：

```bash
unzip -q app.ipa -d /tmp/ipa-check
APP=$(find /tmp/ipa-check/Payload -maxdepth 1 -name '*.app' -type d | head -1)
xcrun assetutil --info "$APP/Assets.car" | rg 'AssetType|Name|PixelWidth|PixelHeight|RenditionName'
/usr/libexec/PlistBuddy -c 'Print :CFBundleIcons:CFBundlePrimaryIcon' "$APP/Info.plist"
```

如果只检查源码，容易把“配置写了”误认为“资源进包了”；如果只看调试基座，又容易把开发链路的 easycom 能力误认为正式导出也具备。最终包的文件和 bundle 才是发布前最可靠的证据。

## 十六、当前验证边界

本轮已用 HBuilderX CLI 重新导出，确认全局组件定义、搜索框样式和本地资源进入 `unpackage/resources`；`vue-tsc`、ESLint 和 81 个测试通过。iOS 图标资源已生成 18 张不透明尺寸图，并写入 manifest。

新的 IPA 还没有安装到实体 iPhone 上，因此组件渲染、搜索框交互、桌面图标刷新和 iOS 缓存仍需真机验收。构建产物检查通过，不等于真机发布门已经完成。

但工程上仍然要区分两件事：

- **本地资源验证通过**：说明构建链和产物方向正确；
- **新 IPA 真机验证通过**：说明云打包、签名、安装和设备 WebView 全链路都正确。

只有重新云打包并安装修复后的新 IPA，确认首页和所有底部导航页面都能显示正文，才算完成最终闭环。把这个边界写清楚，比看到一次“编译成功”更可靠。

## 结语

这次问题最值得记住的不是某一行配置，而是排查方法：当 Debug 正常、正式包异常时，不要先假设是业务数据；当日志说“编译成功”时，也不要默认产物就是正确版本。

“只有底部导航”这个现象本身已经提供了线索：原生壳加载了，但 WebView 业务资源可能没有执行。沿着这个线索检查 IPA 内的 `app-service.js`，很快就能看到 Vue 2/Webpack 与 Vue 3/Vite 的错配，最终再回到 HBuilderX 的项目识别规则，找到 `3` 与 `"3"` 这个真正的开关。

把证据、边界和验证顺序记录下来，下一次遇到类似问题，就不需要靠不断试包来碰运气了。
