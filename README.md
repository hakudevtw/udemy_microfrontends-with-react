Course Link - https://www.udemy.com/course/microfrontend-course/

Site: https://d27xpugoy49gze.cloudfront.net

## 專案需求

- 網路上的一些其他見解 (以 React 為例，可以參考就好)
  - 用 Redux 在 Micro Frontend 之間共享狀態
  - Container 必須使用 Web Components
  - 每個 Micro Frontend 可以是一個能直接被其他 App 引用的 React Component

### Child & Child

- 不能有任何耦合
  - 否則單一專案的更新，會導致另一個耦合專案進行更新
  - 直到有一天可能會沒有人知道 React 怎麼用，也不會改
- 不會共享 functions/objects/classes 等等
- 不會共享 state
- 允許在 Module Federation System 之間共享 Libraries

### Container & Child

- 盡可能不要有任何耦合
  - 就算需要某種最小程度的溝通方式，但 container 不會受到 child 的影響
  - 溝通方式會使用最基礎的模式 Ex. Callback、Event Structures 等等
- 透過 `mount` function 而不是 react component 這種框架限定的東西，我們不會限制任何框架的使用

### Styling

- 不同專案的 CSS 不能互相影響，需要為 Scoped 的方式

### Version Control

- Version Control 的方式不應該影響專案
- 可以自行選擇是否要使用 Separate Repo 或是 Monorepo

### Production & Deployment

- Container 可以決定要用某個版本的 Micro Frontend (或說 Child Version)
- 選擇一: 永遠使用最新版本的 Micro Frontend (Container 不用 Redeploy)
- 選擇二: 使用指定版本的 Micro Frontend (Container 修改時需要 Redeploy)
- Ex. Marketing Project 只有在某個特定時間點才會想要使用最新的版本

## 實作

### Config

- 在察看 network 的時候，載入 library 時可能會發現好像沒有載入重複資源 (還未設定 shared)
  - 以下 兩個檔案看起來是不同的 Library 沒有重複，但實際上這個檔案名稱是隨機的
    - vendors-node_modules_react-dom_index_js.js
    - vendors-node_modules_material-ui_core_esm_Box_Box_js-node_modules_material-ui_core_esm_Button-5a9980.js
  - 也就是我們在設置 shared 之前，裡面仍然同時有 React 和 ReactDOM，檢視檔案內容查詢 react 就能發現了
- shared
  - 有些情況我們想要指定 shared 的資源，例如指定特定版本或設定
  - 但很多時候我們希望 webpack 幫我們自己發現哪些能共用，可以改引用 packageJson.dependencies (不需要轉陣列)

### Deployment

- child app 的 remoteEntry.js URL 必須讓 container 在【build time】就知道! 畢竟在跑的時候需要知道資源在哪裡才能跑
- 要確保我們使用的部屬服務能獨立部屬各個 micro frontend
- 要小心 caching 的問題 (remoteEntry.js)
- 此專案部屬到 AWS S3
  - 使用者導覽到某個 URL 時，會從 AWS CloudFront (CDN) 請求資源
  - AWS CloudFront 會知道要從哪個 S3 bucket 取出檔案
    - container:index.js -> container:main.js -> marketing:remoteEntry.js -> marketing:main.js

### CICD (Github Actions)

- workflow (distinct for each sub project) - container
  - Triggered when code is pushed and commit contains changes to the container folder
  1. 切到 container 資料夾
  2. 下載依賴
  3. 透過 webpack 建立 production build
  4. 將 build result 上傳到 AWS S3

### AWS S3 Bucket

- 預設情況下，所有上傳的資料是私密的，但這裡因為放的是 host files，當然希望他公開
- 到 properties 中 enable static website hosting
  - index document - index.html (這裡會被後面 Cloudfront.. 做的設定覆蓋)
- 到 permissions 中把 Block public access 全部關掉 (跳出的提示可以忽略因為公開內部的資源是我們需要的)
- 到 permissions 中設定 Bucket Policy，點選 Edit 後點選 Generate Policy 開啟分業進行資料填寫生成 Policy
  - Type: S3 Bucket Policy
  - Effect: Allow
  - Principle: \*
  - Actions: GetObject
  - Amazon Resource Name: 到原頁面複製 Bucket ARN 貼過來 + "/\*"
    - `{Bucket ARN}/*`
  - Add Statement -> Generate Policy -> Copy
  - 貼進去原頁面的 Policy -> Save Changes
- 我們不會直接從 Bucket 中取用資源，而是會透過 Amazon CloudFront (CDN) 取用

### Cloudfront Distribution Setup

- Distribution - 一些我們想要公開外部的檔案
- 另開一頁處理 Cloudfront (保留原本 Bucket 的 Tab 等等會用到)
  - Create a CloudFront distribution
    - Origin Domain Name: 選擇剛剛建立的 S3 Bucket
    - Default cache behavior 的 Viewer protocol policy: Redirect HTTP to HTTPS
    - Create Distribution (前面沒提到的都保留預設值就好)
  - 點選剛剛的 Distribution 並在 General 中的 Settings 點選 Edit (Remapping error)
    - Default root object: `/container/latest/index.html`
  - 到 Error pages 中點選 Create custom error response
    - HTTP error code: 403: Forbidden
    - Customize error response: yes
      - Response page path: `/container/latest/index.html`
    - HTTP Response Code: 200: Ok
  - 回到 General 看到的 Distribution domain name 就是接下來要請求的路徑

### Github Actions & AWS

- AWS_ACCESS_KEY_ID & AWS_SECRET_ACCESS_KEY 用來存取我們的 AWS 帳號
  - 到 AWS Console 中透過 IAM 生成
    - Users -> Create user
      - 填寫名字
      - Next
      - Permissions options: Attach policies directly
        - 比較好的方式應該要限制這個使用者能夠存取的特定 Bucket
      - Permissions policies: AmazonS3FullAccess & CloudFrontFullAccess
      - Next
      - Create user
    - 點進去剛剛建立的 user 並到 Access key 欄位點選 Create access key
      - Use case: Command Line Interface (CLI)
      - Check confirmation
      - Next
      - Copy secret key
- AWS_DEFAULT_REGION
  - 去 S3 中找到 Bucket 並從最後面取得 Region (Ex. ap-southeast-2)
- 到 github 設定 action yml 中所需要的 secrets
  - Settings -> Secrets and variables -> Actions
  - Repository secrets -> New repository secret
  - 建立 AWS_S3_BUCKET_NAME / AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY

### Trouble Shooting

- 成功部屬到並嘗試瀏覽 URL 時，會發現 main.js 正被嘗試從 bucket root 取得
  - 我們實際上是把東西推到了 `container/latest/` 底下，理所當然沒辦法從 root 取得
  - 設置 `output.publicPath`讓 HTMLWebpackPlugin 在引入 output file 時加上 `/container/latest/`

### Caching

- 在我們更新 Deployment 並重整頁面後，會發現 HTML 並沒有被更新 (後面 remoteEntry.js 也會遇到一樣的問題)
- 問題出在 Cloudfront 運作方式，在更新 Distribution 時，Cloudfront 只會看有沒有增加刪除檔案，但不會主動去查看檔案是否有變動
  - 只有 HTML 並沒有加上 hash，因此並不會知道裡面有所變更
- 到 AWS Cloudfront 進行 Invalidation (手動)
  - 點選當個 Distribution 並進到 Invalidations Tab
  - Create invalidation -> `/container/latest/index.html`
  - Create Invalidation
- 到 workflow 裡面自動化 Invalidation 流程
  - 這裡會發現 env 被重複設定，的確可以把它移動到最上面讓大家都能存取，但同時也代表其他不需要的動作也能存取
  - 因此還是建議在需要的地方設定重複就好

## Styling

- 在 Micro Frontend 的實作中很常遇到 Local 端沒問題但部署上去畫面卻出現問題的狀況
  - Ex. 在 SPA 的專案底下，頁面 A 使用了 CSS A 沒有問題，跳轉到頁面 B 之後載入了 CSS B，重新跳轉回頁面 A 時，頁面 A 就會吃到 CSS B 的樣式，導致出現樣式衝突
- 如果是為了統一 Library 所帶來的樣式而全部都要求用同一個版本，會導致後面升級時大家都要升級，違反了一開始 Micro Frontend 的初衷

### Scoping CSS

- CSS-in-JS Library - 透過 JS 動態生成 unique class 並加到元素上
- Vue 或 Angular 內建的 component style scoping
- Namespace 所有的 CSS - 在 root element 上加上一層 class，並在選擇棄前面都加上那層 class
- CSS Modules

### CSS-in-JS Libraries

- 當我們在 sub-projects 中使用相同的 CSS-in-JS Library 時，很有可能出現 class name 衝突，就有可能導致上述範例遇到的問題
- 以 MUI 的 `makeStyles()` 為例
  1. `makeStyles({ heroContent: { padding: '20px' } })`
  2. 取得 JS Object 如 `{ heroContent: 'makeStyles-heroContent-2' }`，這裡的 Value 為某種隨機生成的 class name，會用來加在畫面上的某個 Element
  3. 與此同時會產生一個對應到上面 class name 的 CSS
- 會遇到衝突通常會是在打包成 Production 的階段
  - 在 Production 時，我們通常會希望將這些 CSS class 或 rule 打包成一隻獨立的 CSS Stylesheet
  - Extract 的過程中通常會希望避免有太長的 class name 來縮減檔案大小
  - 因此不像 Development 中隨機生成一組 class 的方式，而是產生如 `jss1`、`jss2` ... 這種名稱 (重點是他不是真的隨機！)
  - 在 Child Project 之間的 Build Process 是分開的，因此很容易的就會產生相同的 class name，導致在 production 組合時出現 class name 之間的相互衝突
- 通常會提供一些解決方式，如 MUI 就提供了 `generateClassName()` 讓我們設值 `productionPrefix`，也就是在 production 產出的 class name 修改被加上的 prefix 來避免和其他 production build 的衝突

  ```js
  // App.js
  createGenerateClassName({ productionPrefix: "ma" });
  <StylesProvider generateClassName={generateClassName}>

  // Production
  <div class="ma1"/> // 而不是 js1
  ```

- 想到 Tailwind 好像也可以做類似的事情 XD
- 可以的話盡量都加，避免未來出現的衝突，命名上如果能統一其實也蠻方便看得出來是哪個專案來的 class name

## Navigation

- 有很多種實作方式，要視專案需求去設計
- 在這裡會很明顯發現會需要讓 Child App 或與 Container 間做資料的交換
  - 要記得實作的時候要盡可能的 Generic，避免互相限制使用的 Library 或 Version 甚至不同的實作方式
  - 一個 Routing Solution 的改變不應迫使調整其他 App 的寫法

### Requirements

- Container 和 SubApps 都需要某些 Routing Features
  - Navigate 的時候需要 Container 的 Routing Logic 顯示不同的 SubApps
  - SubApps 中本身會需要自己的 Routing Logic 來做內部的 Navigation
  - 並非所有的 SubApps 都需要 Routing
- SubApps 需要有能力增加新的頁面
  - 新增頁面或 Route 的同時，不需重新 Deploy Container
  - 也就是 Container 可能只負責決定要顯示哪個 SubApp，剩下的 Routing 由 SubApp 控制
- 畫面上很有可能需要同時顯示超過兩個的 Micro Frontend
  - 例如被拆出來成一個獨立 Micro Frontend 的 Sidebar
- 不應建立自己的 Routing System，應該用現有的 Libraries 做處理
  - Ex. react-router, vue-router, angular router
  - 加入一點 Code 做微調是允許的
- Navigation Features 需要同時存在於 Hosted Mode 或 Isolation Mode
  - 也就是開發時也要能確實知道自己位於哪個 Path 上

### Solution

1. Container 和 SubApp 各自能有自己使用的 Routing Library，就算交換資訊也會透過 Generic 的方式
2. Container Routing 只負責決定要顯示哪些 MicroFrontend，而 SubApp MicroFrontend (Ex. Marketing) 會決定要顯示哪個頁面
   - 如果要同時顯示多個 MicroFrontend，也可以透過相同透過 container app 藉由判斷 root path 顯示不同組合的方式處理

### How Routing Library Works

- 主要分為兩個部分 - History Object 和 Router

#### Router

- 根據使用者造訪的路徑，決定要顯示哪個畫面

#### History Object

- 用來知道使用者目前造訪的路徑，並隨瀏覽進行路徑的修改
- 畫面上的 `Link` 的作用域會自動去看離他最近的 Router History
- 通常會分成三種不同的 History
  - Browser History - 透過 URL 知道使用者目前造訪的 path，也就是 domain 之後的部分，並將這些資訊送去給 Router 決定顯示畫面 (Ex. `react-router-dom` 的 `<BrowserRouter/>`)
  - Hash History - 查看 URL `#` 後面的部分
  - Memory/Abstract History - 把當前路徑資訊存於 memory 也就是 code 當中，完全不會透過 Address URL 來得知使用者所在路徑
- 而在建立 Router 的時候，需要告訴 Library 要使用哪一種 History
- 不同 Library 對於 History 的實作方式並不相同，如果都使用 Browser History，可能導致透過不同的實作方式同時嘗試修改 URL
  - 可能導致 Library 之間產生某些更新時機點的 Race Condition，甚至是更新的方式不同
  - 因此不太會共享操作同一個 Browser History
- 在 MicroFrontend 的實作中，最常見的方式為
  - 在 Container 中使用 Browser History，直接讀取和操作 URL
  - 在 SubApps 中使用 Memory History，各自透過複製一份 URL 到 memory 中進行操作，Navigate 的時候也是操作 Memory 的而不是直接操作 URL

### Syncing Histories

- 首先把 Marketing App 改使用 Memory History，此時發現兩種情境的不同狀況
  - 從 `localhost:8080/` 點選 Marketing App 中雖然畫面改變了，但 URL 仍停留在原點 (`/`)
    1. 進入 `localhost:8080/` 後同時產生了兩份 History，Container App 的 Browser History (`/`)，和 Marketing 的 Memory History (`/`)
       - Browser History 的初始值會看 URL
       - Memory History 的初始值永遠是 `/`
    2. 當我們點選 Marketing App 中的 `pricing` 連結時，Marketing 的 Memory History 變成了 `/pricing`，而 Marketing App 中的 Router 也接收到了這個資訊並正確的更新了畫面
       - 但可想而知，Container App 的 Browser History 並不知道這件事，因此還停留在初始值 (`/`)
  - 從 `localhost:8080/pricing` 進入畫面，URL 是對的但 Marketing App 仍顯示 Landing Page 而非 Pricing Page
    1. 進入 `localhost:8080/pricing` 後同時產生了兩份 History，Container App 的 Browser History (`/pricing`)，和 Marketing 的 Memory History (`/`)
       - 理所當然 Marketing App 顯示的會是對應到 `/` 的 Landing Page 而非對應到 `/pricing` 的 Pricing Page
    2. 此時如果點選 Marketing App 的 Pricing，更新後的 Memory History 會促使畫面的更新，但當我們重新透過 Navigation 切換 URL 時，仍會遇到一樣的狀況
- 因此我們很快會遇到兩大問題，如何在 MicroFrontend 之間溝通，並如何同步 Navigation
  - 當使用者點選了 Container 的 Link 時，需要將資訊傳下去給 Marketing 去更新他的 Memory History，進而更新顯示畫面
  - 當使用者點選了 Marketing 的 Link 時，需要將資訊傳上去給 Container 去更新他的 Browser History，進而更新顯示畫面

### Communication

- 為了避免專案中的耦合，會使用基礎的方式進行資料交化
  - Ex. Events、Callbacks
- 以 Navigation 而言，就是將 `onNavigate` 從 container 傳下去給 marketing，讓 marketing 使用 memory history 進行 navigation 時，呼叫 `onNavigate` 通知 container 更新 browser history
  - 過程中因為互相偵測改變可能導致無限輪迴，要特別注意，可以透過

## 了解 publicPath 設置

- 前情提要
  - marketing 中 `webpack.prod.js` 所設置的 `publicPath` 是為讓 marketing 的 `remoteEntry.js` 能找到其對應的資源
  - container 中 `webpack.prod.js` 所設置的 `publicPath` 則是為了能讓 HTMLWebpackPlugin 能在 `index.html` 引入正確的資料路徑
- 當我們把兩個頁面設在 `/auth/signin` 和 `/auth/signup/` 時，造訪後會發現 404 找不捯 `/auth/main.js`
  - 當引入資源如 `<script src="main.js"/>` 時，瀏覽器會常識從當前的 domain + path 去尋找 `main.js`
    - 也就是當瀏覽 `localhost:8082/auth/signup` 時，瀏覽器會當作要從 `localhost:8082/auth/` 底下嘗試載入 `main.js`
    - 在最後面加上 `/` 的話則會從 `localhost:8082/auth/signup/` 底下嘗試載入 `main.js`
  - 而我們的資源實際上是放在 `localhost:8082/` 底下，理所當然找不到
- 如果在 development 的時候也把 `output.publicPath` 設為 `/` 呢？
  - 這樣在 Auth Isolation 載入資源時，的確會正確從 `localhost:8082/` 尋找 `main.js`
  - 但很快會發現這樣的做法不太適用於 Micro Frontend 的架構中
    1. 在 `localhost:8080` 載入 container
    2. container 會去 `localhost:8082` 找到 auth 的 `remoteEntry.js` 並了解該如何載入資源
    3. 這時我們所設置的 `publicPath: "/"` 也會影響到這隻 `remoteEntry.js`，因此會從 `/` 去嘗試載入 auth 的 `main.js`
       - 但回顧上面的前情提要，現在的 domain + path 是 `localhost:8080`！因此它實際上載入的是 container 的 `main.js` 而不是 auth 的！
- 正確的修改方式是在 auth development config 中將 `publicPath` 改為完整的 URL，也就是 `http://localhost:8082/`
- 那為什麼 marketing 沒有設置 `publicPath` 不會有問題？
  - 因為沒有設置的情況下，`remoteEntry.js` 會從和他自己相同的 domain 底下載入資源
  - 也就是當 container 從 `localhost:8081` 載入 marketing 的 `remoteEntry.js` 時，`remoteEntry.js` 也會從 `localhost:8081` 載入其他 marketing 資源
- 因此習慣性在建立 child app 的 development 環境時，仍會設置 `publicPath` 來確保正確的載入路徑

## Authentication

- auth app 本身不負責權限控管、路由限制或是知道使用者是否登入的資訊
- 處理 Authentication 的方式有兩種
  - 所有 Authentication 集中控制於 Container，並將資訊告訴 Sub Apps
  - 所有的 App 都有一些 Authentication 的 Code，但這樣比較容易寫很多重複的 code
