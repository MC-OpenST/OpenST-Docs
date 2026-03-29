# 1. 档案馆技术栈  

档案馆是用vue+tailwind css编写的，通过检索database.json进行卡片组合，并显示于前端  

档案馆核心代码树如下：  
```markdown
.root
  > archive.html
  > scripts
      - auth.js
      - build.js
      - config.js
      - logic.js
      - main.js
      - ui.js
  > tailwind.js
  > data/database.json
  > archive
      - 各种各样的稿件
          * example.litematic
          * info.json
          * preview.png
          * preview.webp
  > auth-callback.html
```  

## 1.1 数据流架构  

上传逻辑会将投影，预览图，简介自动打包成一个特殊的数据zip，解压后将获得以下数据结构：  
```markdown
> OpeST_File
    - 稿件名称
        > example.litematic
        > preview.png
        > info.json
```
> 实际上不一定是litematic，可以是zip或者rar等压缩包。预览图也不一定是png而是jpg  

过审核后，由审核员手动将“稿件名称”文件夹给送入archive文件夹中。下一步由github action执行  
action负责启动script/build.js，完成的工作是将archive文件夹所有的文件和子文件递归，并通过每个单独稿件的info.json作为基本数据写入database.json内  
```json
{
    "id": "sub-1772764832049",
    "name": "【1.16+】0t 堆肥桶可访问打包",
    "author": "ZeroTwo",
    "tags": [
        "1.16+",
        "可访问打包"
    ],
    "description": "### 🚀 机器概览\n- **核心功能**: \n堆积 27 物品后进行打盒换盒，100% 收集率。（推荐用于不可堆叠分类打包）\n\n- **适用版本**: Java 1.16+\n- **投影创建者**：ZiYe__ZeroTwo \n- **投影创建时间**：2026-03-06\n\n### 📖 使用说明\n1. 在非原版特性端使用时，请先测试机器能否正常工作后再进行实装。\n\n> 提示：本机器应横向堆叠使用。",
    "folder": "【1.16+】0t 堆肥桶可访问打包",
    "preview": "preview.webp",
    "filename": "0t堆肥桶可访问打包机.litematic",
    "submitDate": "2026-03-16T02:33:06.550Z"
}
```
<small>一个info.json的示例</small>  

**info.json的每个定义如下：**
- id = 从1970年到现在的时间
- name = 稿件名称
- author = 作者名
- tags = 标签，可多选
- description = 简介，前端支持markdown渲染
- folder = 位于archive哪个文件夹下
- preview = 预览图
- submitDate = 上传时间

而最终生成的database.json则是这样的：  
```json
    {
        "id": "CMIS-V5",
        "name": "【1.21+】CMIS-V5",
        "author": "Civic Dude",
        "tags": [
            "1.21.x",
            "全物品仓库",
            "多物品仓库"
        ],
        "description": "## **机器介绍：** \nCMIS 是一款以生存建筑为核心设计的全物品，融合了简单与复杂的设计理念，打造出一个功能齐全的全物品，并针对以下方面进行了优化：\n- 分拣速度（平均每小时 55,000 件物品）\n- 高效率\n- 生存友好  \n\n**适用版本**：Java 1.21+\n**投影创建者**：CivicDude\n**投影创建时间**：2026-01-21\n\n### **该全物品的主要组件包括：**\n\n1. 高速分盒器\n\n2. 白名单，用于分离散装、MIS 和可堆叠物品，并区分每种物品的部分包装和完整包装\n\n3. 打包机，用于压缩白名单物品（9:1 配方），并将这些物品的部分包装存储起来，直到产出满盒\n\n4. 自制 MIS 单片，允许存储盒装或散装 MIS 物品（混储），完全并行化，具有溢出保护功能，并在 32 GT 时钟上运行\n\n5. 盒子分类大宗单片，溢出的物品送入远程大宗\n\n6. 可从远程大宗自动 / 手动补货\n\n7. 不可堆叠分类\n\n以及其他 6 个外部组件  \n## **特别鸣谢：**\n- TisUnfortunate\n- Andrews54757",
        "preview": "archive/CMIS-V5/preview.webp",
        "filename": "CMIS-V5.0.litematic",
        "sub_id": "sub-1772730841971"
    },
```

最显著的不同是id，database.json的id不等于info.json的id。为此如果维护时请注意这方面  

build.js也负责了将图片转化为webp格式的任务，这点是为了让前端压力减小。转换的外部库是sharp 0.34.5  

## 1.2 上传稿件+登录逻辑  

目前档案馆的登录架构是通过github登录，使用了GitHub OAuch作为唯一的登录方式  
> 这点需要未来做修改，由于github在国内的不稳定性，为此此段落随时可能修改  

github登陆后，获取动态token，以便完成自动提交issue  
这部分的核心代码
```markdown
.root
    > upload
        > upload.html
        > upload.js
    > scripts
        > auth.js
    auth-callback.html
外部组件：
cloudlare workers
```  

登录逻辑+上传稿件的原代码片段：  
```js
export default {
    async fetch(request, env) {
        const url = new URL(request.url);
        if (request.method === "OPTIONS") return handleCORS();

        try {
            // --- [1] OAuth 令牌交换 ---
            if (url.pathname === '/api/exchange-token') {
                const code = url.searchParams.get('code');
                const res = await fetch('https://github.com/login/oauth/access_token', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json', 'Accept': 'application/json' },
                    body: JSON.stringify({ client_id: CLIENT_ID, client_secret: CLIENT_SECRET, code: code })
                });
                const data = await res.json();
                if (data.access_token) {
                    const userRes = await fetch('https://api.github.com/user', {
                        headers: { 'Authorization': `token ${data.access_token}`, 'User-Agent': 'OpenST-Portal' }
                    });
                    data.user = await userRes.json();
                }
                return new Response(JSON.stringify(data), { headers: { ...getCORSHeaders(), "Content-Type": "application/json" } });
            }

            // --- [2] 权限校验 ---
            if (url.pathname === '/api/check-admin') {
                const token = request.headers.get('Authorization')?.replace('Bearer ', '');
                if (!token) return new Response("Unauthorized", { status: 401, headers: getCORSHeaders() });
                const ghRes = await fetch(`https://api.github.com/repos/${GH_REPO}`, {
                    headers: { 'Authorization': `token ${token}`, 'User-Agent': 'OpenST-Portal' }
                });
                const repoData = await ghRes.json();
                const isAdmin = repoData.permissions?.push === true;
                return new Response(JSON.stringify({ isAdmin }), { headers: { ...getCORSHeaders(), "Content-Type": "application/json" } });
            }...
```

```js
// --- [4] 投稿中继 ---
            if (request.method === 'POST' && url.pathname === '/') {
                const fd = await request.formData();
                const zipFile = fd.get('zip');
                const previewFile = fd.get('preview');
                const name = fd.get('name');
                const photoFd = new FormData();
                photoFd.append('chat_id', CHAT_ID); photoFd.append('photo', previewFile); photoFd.append('caption', `📦 新投稿：${name}`);
                await fetch(`${TG_API_BASE}/sendPhoto`, { method: 'POST', body: photoFd });
                const docFd = new FormData();
                docFd.append('chat_id', CHAT_ID); docFd.append('document', zipFile);
                const docRes = await fetch(`${TG_API_BASE}/sendDocument`, { method: 'POST', body: docFd });
                const docData = await docRes.json();
                const fileInfoRes = await fetch(`${TG_API_BASE}/getFile?file_id=${docData.result.document.file_id}`);
                const fileInfo = await fileInfoRes.json();
                const downloadUrl = `${url.origin}/dl/${fileInfo.result.file_path}?fn=Archive_${encodeURIComponent(name)}.zip`;
                return new Response(JSON.stringify({ success: true, filePath: fileInfo.result.file_path }), { headers: { ...getCORSHeaders(), "Content-Type": "application/json" } });
            }

            // --- [5] 下载代理 ---
            if (url.pathname.startsWith('/dl/')) {
    const path = url.pathname.replace('/dl/', '');
    const customFileName = url.searchParams.get('fn'); // 获取 ?fn= 后面的名字

    const response = await fetch(`https://api.telegram.org/file/bot${BOT_TOKEN}/${path}`);
    
    // 必须创建一个新响应对象，因为原始 response 的 headers 是不可变的
    const newRes = new Response(response.body, response);
    
    // 覆盖 CORS
    newRes.headers.set("Access-Control-Allow-Origin", "*");
    
    if (customFileName) {
        // 关键：使用 filename*=UTF-8'' 语法强制浏览器识别编码并作为附件下载
        // 这会覆盖 Telegram 默认的 file_x 命名
        const safeName = encodeURIComponent(customFileName);
        newRes.headers.set(
            "Content-Disposition", 
            `attachment; filename*=UTF-8''${safeName}`
        );
    } else {
        newRes.headers.set("Content-Disposition", "attachment; filename=\"OpenST_File.zip\"");
    }
    
    // 强制移除 Telegram 可能干扰文件名的 Content-Type
    newRes.headers.set("Content-Type", "application/octet-stream");

    return newRes;
}
            if (url.pathname === '/health') {
                const startTime = Date.now();
                
                // 异步抓取 CF 官方状态 (中继避免跨域)
                const cfStatusPromise = fetch('https://www.cloudflarestatus.com/api/v2/status.json')
                    .then(r => r.json())
                    .catch(() => ({ status: { description: "Unknown" } }));

                const cfOfficial = await cfStatusPromise;

                return new Response(JSON.stringify({
                    status: 'Operational',
                    region: request.cf?.colo || 'Edge', // 节点代码，如 HKG, SHA
                    latency: Date.now() - startTime,
                    upstream: cfOfficial.status.description, // CF 官方状态
                    timestamp: startTime
                }), { 
                    headers: { ...getCORSHeaders(), "Content-Type": "application/json", "Cache-Control": "no-store" } 
                });
            }...
```

管理员检测是通过是否有对仓库拥有push的权限，同时由于将client secret和bot token储存于前端并不安全，为此选择了存在workers里  

登录实现方式：  
1. 回调页 (auth-callback.html) 抓 code 丢给 Worker
2. Worker (api/exchange-token) 藏好 Secret 换 Token
3. 本地 (auth.js) 存 Token 并定时 7d 清理

### 1.2.1 上传逻辑的具体实现  

获取到了token后，在提交页面内，用户将投影，预览图，简介，标签输入完毕后，打包成一个zip压缩包输送到tg群内部。并通过workers生成一个唯一的下载链接  
而为了防止出现file0,file1.zip这样的情况，workers会中途拦截，并更名zip压缩包的名称  
最后，则是通过workers将固定模板+下载链接+用户token自动提交issue  

> note: archive.html的登录状态是跟upload.html的上传状态是完全同步的

## 1.3 档案馆的标签，本地设备我的收藏，繁简转换  

核心代码文件：
```markdown
.root
    > scripts
        > logic.js
        > ui.js
        > main.js
        > config.js
    > Traditional-Simplefild （我知道我打错了Simplified但是我懒得改）
        > STCharacters.txt
    > archive.html
```

**标签的逻辑：**  
1. 加载config.js的硬编码标签

    ```js
    // config.js
    export const TAG_CONFIG = {
        "非编码存储科技": {
            "全物品单片": ["8箱单片", "10箱单片", "其他单片"],
            "大宗仓库": ["分类打包大宗", "盒分大宗", "四边形大宗"],
            "不可堆叠相关": ["不可堆叠分离", "不可堆叠分类"],
            "多物品相关": ["多物品分类 (MIS)", "多种类潜影盒分类 (MBS)"]
        },
        "潜影盒处理": {
            "潜影盒打包机": ["分类打包", "混杂打包", "自适应打包", "精密打包", "缓存打包", "可访问打包", "比例打包", "堆分打包"],
            "潜影盒拆包机": ["漏斗拆包", "矿车拆包 (Yeeter)", "烧包机"],
            "潜影盒展示": ["无残留展示", "自出盒展示", "常用物品展示", "可反悔展示", "上行展示", "精密展示", "细雪展示", "灵魂沙展示", "堆肥桶展示"],
            "潜影盒分类相关": ["潜影盒分类", "自适应潜影盒分类 (SVAR)", "车头及拐弯设计 (Keygen & U-turn)"],
            "分盒器相关": ["潜影盒分盒器", "理想分盒器 (矿车)"],
            "合并器相关": ["潜影盒合并器", "不满盒临时存储", "分组器 (Grouper)", "配对优化器"],
            "潜影盒填充度分类": [],
            "空盒仓库": []
        },
        "编码存储科技": {
            "编码全物品单片": ["常规编码单片", "矩阵编码单片", "高速编码单片", "其他编码单片"],
            "编码大宗仓库": ["编码大宗", "编码四边形大宗", "远程大宗"],
            "逻辑/传输": ["逻辑电路", "移位寄存器", "转码器"],
            "编码器": ["二进制", "其他编码器"],
            "解码器": ["二进制", "十进制", "十六进制", "强模 (HSS/OSS)", "水道速", "方块事件深度(BED)"],
            "其他组件": []
        },
        "其他杂物": {
            "仓库成品": ["全物品仓库", "多物品仓库", "编码全物品"],
            "进阶组件": ["潜影盒UI", "潜影盒硬盘", "动态大宗"],
            "整流器": ["盒流整流器", "堆分整流器", "矿车整流器"],
            "分流器/黑白名单": [],
            "堆分离/分类": [],
            "无实体输入": [],
            "水道相关": [],
            "地狱门加载器": []
        },
        "生产/合成": ["合成站", "合成器相关"],
        "版本": ["1.21.x", "1.20.x", "1.19+", "1.17+", "1.16+"],
    };
    export const CATEGORIES = Object.keys(TAG_CONFIG);
    ```  

2. logic.js处理标签部分的加载
3. main.js获取到后，转发给前端html

**本地我的收藏逻辑：**  
1. 监听前端
2. 如果点击了前端html的`archive-card`则录入localstorage储存
3. 如果前端引用则从localstorage加载

**繁简转换逻辑：**
- 通过STcharacters字典进行转换，懒得写了自己看代码吗
   ```js
   // 解析字典
   const dictText = await dictRes.text();
   
   // 确保定义了变量
   const fs = [];
   const ft = [];
   
   dictText.split(/\r?\n/).forEach(line => {
       if (!line || line.startsWith('#')) return;
       const parts = line.trim().split(/\s+/);
       if (parts.length >= 2) {
           parts.slice(1).forEach(t => {
               fs.push(parts[0]);
               ft.push(t);
           });
       }
   });
   
   this.dictSArray = Object.freeze(fs);
   this.dictTArray = Object.freeze(ft);
   ```


### 1.3.1 搜索框逻辑+?sub-xxx搜索逻辑
- 懒得写了看代码  


```js
if (query.includes('sub-')) {
    return data.filter(item =>
        item.sub_id && item.sub_id.toLowerCase().includes(query)
    );
}
```


```js
// 搜索框逻辑
if (!query) return true;

// #前缀模式
if (query.startsWith('#')) {
    const tagQuery = query.slice(1);
    if (!tagQuery) return true;
    return item.tags && item.tags.some(t => t.toLowerCase().includes(tagQuery));
}

// 常规模糊匹配
const normQuery = normalizeFn ? normalizeFn(query) : query;
const normName = normalizeFn ? normalizeFn(item.name || "") : (item.name || "").toLowerCase();
const normAuthor = normalizeFn ? normalizeFn(item.author || "") : (item.author || "").toLowerCase();
const normDesc = normalizeFn ? normalizeFn(item.description || "") : (item.description || "").toLowerCase();

const matchText = normName.includes(normQuery) ||
    normAuthor.includes(normQuery) ||
    normDesc.includes(normQuery);

const matchTags = item.tags && item.tags.some(t => t.toLowerCase().includes(query));

return matchText || matchTags;
});
```

总而言之，实际上如果你想添加一个功能，只需修改`main.js`即可，而如果涉及到标签部分则需修改`logic.js`。ui则是`ui.js`。