# 静态网站重构方案

## 一、现状分析

### 当前结构

```
index.html          ← 首页（个人信息 + 高亮展示）
projects.html       ← 项目画廊页
styles/
  home_style.css    ← 首页专用样式
  project_style.css ← 项目页专用样式
scripts/            ← 空目录，无 JS
images/             ← 所有图片
videos/             ← 所有视频
```

### 存在的问题

| 问题               | 说明                                                                      |
| ------------------ | ------------------------------------------------------------------------- |
| **导航栏重复**     | 两个页面有完全不同的导航栏 HTML 和样式，修改一处另一处不同步              |
| **无共享 CSS**     | 两份 CSS 文件各自独立，颜色、字体等基础变量无统一管理                     |
| **固定宽度布局**   | 首页写死 `width: 1000px`，无响应式支持，移动端体验差                      |
| **添加项目成本高** | 添加新项目需手写完整 `<details>` HTML 块，容易出错                        |
| **无语义化**       | `<div id="container">` 等缺乏语义标签（`<header>`, `<main>`, `<footer>`） |
| **ID 选择器泛滥**  | 大量使用 `#id` 选择器，样式优先级难以管理                                 |

---

## 二、三套可选方案

### 方案 A：原生 HTML/CSS/JS 整理（最小改动）

**思路**：不引入任何构建工具或框架，仅重组文件结构、提取共享部分。

#### 改动点

1. **提取公共 CSS 文件**
   - 新建 `styles/shared.css`，存放 CSS 变量、reset、导航栏、排版等公共样式
   - `home_style.css` 和 `project_style.css` 仅保留各自页面的差异样式

   ```
   styles/
     shared.css          ← 新增：变量 + reset + 导航栏 + 公共布局
     home_style.css      ← 精简：仅首页特有样式
     project_style.css   ← 精简：仅项目页特有样式
   ```

2. **统一导航栏**
   - 两个页面使用同一套导航栏 HTML 结构和样式
   - 用 JS 动态插入导航栏，避免手动同步：

   ```
   scripts/
     components.js       ← 新增：动态生成 navbar/footer
   ```

3. **用数据驱动项目列表**
   - 新建 `data/projects.json`，将项目信息（标题、描述、图片列表、链接）集中管理
   - 用 JS 读取 JSON 并动态渲染 `<details>` 列表
   - 添加新项目只需编辑 JSON 文件

   ```json
   [
     {
       "title": "C++ Game Design",
       "github": "https://github.com/lby06/1340Group2",
       "description": "...",
       "images": ["images/p1_1.png", "images/p1_2.png", "images/p1_3.png"]
     }
   ]
   ```

4. **引入 CSS 变量**

   ```css
   :root {
     --color-primary: #1877f2;
     --color-bg-dark: #494949;
     --color-text: #2e2e2e;
     --max-width: 1000px;
   }
   ```

5. **语义化 HTML**
   - `<div id="heading-toolbar">` → `<header>`
   - `<div id="container">` → `<main>`
   - `<div id="left-sidebar">` → `<aside>`

**优点**：零依赖，无构建步骤，GitHub Pages 直接部署，学习成本最低  
**缺点**：JS 动态渲染导航栏对 SEO 不友好（个人网站影响小）；项目复杂后管理仍较原始

---

### 方案 B：Jekyll 静态站点生成器（GitHub Pages 原生支持）

**思路**：GitHub Pages 内置 Jekyll 支持，无需额外 CI/CD 配置。用模板和数据文件消除重复。

#### 改动点

1. **目录结构重组**

   ```
   _layouts/
     default.html        ← 公共 HTML 骨架（含导航栏）
   _includes/
     navbar.html          ← 导航栏组件
     footer.html          ← 页脚组件（可选）
   _data/
     projects.yml         ← 项目数据
   _sass/
     _variables.scss      ← 颜色、字体变量
     _navbar.scss         ← 导航栏样式
     _home.scss           ← 首页样式
     _projects.scss       ← 项目页样式
   assets/
     css/
       main.scss          ← 入口，@import 所有 _sass 文件
     images/
     videos/
   index.html             ← 使用 layout: default
   projects.html           ← 使用 layout: default + 循环渲染 _data/projects.yml
   ```

2. **模板化导航栏**
   - `_includes/navbar.html` 写一次，所有页面自动复用
   - 当前页高亮通过 `page.url` 判断

3. **数据驱动项目列表**

   ```yaml
   # _data/projects.yml
   - title: C++ Game Design
     github: https://github.com/lby06/1340Group2
     description: A terminal-based game built with C++
     images:
       - images/p1_1.png
       - images/p1_2.png
       - images/p1_3.png
   ```

   在 `projects.html` 中用 Liquid 模板循环渲染：

   ```html
   {% for project in site.data.projects %}
   <details class="projects">
     <summary>{{ project.title }}</summary>
     ...
   </details>
   {% endfor %}
   ```

4. **SCSS 变量和模块化**
   - Jekyll 原生支持 SCSS，可以使用变量、嵌套、mixin

**优点**：GitHub Pages 零配置支持；模板消除所有重复；SCSS 开箱即用；添加项目只需编辑 YAML  
**缺点**：需要学习 Liquid 模板语法和 Jekyll 目录约定；本地预览需安装 Ruby + Jekyll

---

### 方案 C：Astro 框架 + GitHub Actions 部署

**思路**：用现代前端框架 Astro 构建，组件化开发，最终输出纯静态 HTML。

#### 改动点

1. **目录结构**

   ```
   src/
     components/
       Navbar.astro
       ProjectCard.astro
       Gallery.astro
     layouts/
       BaseLayout.astro
     pages/
       index.astro
       projects.astro
     data/
       projects.json
     styles/
       global.css
   public/
     images/
     videos/
     docs/
   astro.config.mjs
   ```

2. **组件化**
   - 导航栏、项目卡片、图片画廊各自封装为 `.astro` 组件
   - 每个组件包含自己的 HTML + scoped CSS

3. **GitHub Actions 自动部署**
   - 推送到 main 分支时自动 `astro build` 并部署到 GitHub Pages

**优点**：组件化开发体验好；scoped CSS 无冲突；TypeScript 支持；生态丰富  
**缺点**：引入 Node.js 构建步骤；需配置 GitHub Actions；学习成本最高

---

## 三、方案对比

| 维度         | 方案 A（原生整理）       | 方案 B（Jekyll）     | 方案 C（Astro）   |
| ------------ | ------------------------ | -------------------- | ----------------- |
| 学习成本     | ★☆☆                      | ★★☆                  | ★★★               |
| 添加项目难度 | 编辑 JSON                | 编辑 YAML            | 编辑 JSON         |
| 消除重复     | JS 动态插入              | 模板 include         | 组件 import       |
| 构建步骤     | 无                       | GitHub Pages 自动    | 需 GitHub Actions |
| 本地预览     | 双击 HTML 或 Live Server | 需安装 Ruby + Jekyll | 需 Node.js        |
| 样式管理     | CSS 变量                 | SCSS 变量 + 模块     | Scoped CSS        |
| SEO          | JS 渲染不利              | 纯静态 HTML，最优    | 纯静态 HTML，最优 |
| 可扩展性     | 低                       | 中                   | 高                |

---

## 四、推荐

**对于当前项目规模（2 页 + 4 个项目），推荐方案 B（Jekyll）**，理由：

1. GitHub Pages **原生支持**，零额外配置即可部署
2. 模板 include 完美解决导航栏重复问题，且输出纯静态 HTML
3. `_data/projects.yml` 数据驱动，添加新项目只需追加几行 YAML
4. SCSS 支持让样式管理更规范
5. 学习成本适中，Liquid 模板语法简单

如果不想引入 Ruby 依赖，**方案 A** 也是务实的选择——用最少改动解决最突出的问题（重复导航栏 + 项目数据管理）。

---

## 五、无论选哪个方案都建议做的改进

以下改进与方案无关，应当优先执行：

1. **响应式布局** — 去掉 `width: 1000px` 写死值，用 `max-width` + 媒体查询适配移动端
2. **语义化标签** — `<header>`, `<main>`, `<aside>`, `<footer>` 替代纯 `<div>`
3. **CSS 变量** — 统一管理颜色、字体、间距
4. **统一两页导航栏风格** — 目前首页是顶部横向栏，项目页是侧边 3D 菜单，风格割裂
5. **为每个项目添加文字描述** — 符合 AGENTS.md 中的需求
