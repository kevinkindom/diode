---
title: "前端使用K8S搭建测试环境 - 基础环境搭建"
date: 2020-06-22T13:23:21+08:00
draft: true
---

我们先制作上一篇文章中提及的两个镜像。

### frontend_toolkit 镜像

```Dockerfile
FROM ubuntu:latest

# 将nodejs复制到容器下，防止因网络原因导致镜像打包慢或失败的问题
# 版本按需选择
COPY nodejs_setup_13.x.sh /home/

# 安装nodejs
RUN bash /home/nodejs_setup_13.x.sh

# 安装基础工具
RUN apt-get install -y git nodejs curl

RUN curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add -

RUN echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list

RUN apt-get update && apt-get install -y yarn vim
```

### 服务基础镜像
```Dockerfile
FROM nginx:latest

WORKDIR /var/wwwroot

# 将nginx配置文件复制到容器下
COPY ./gaia/nginx/* /etc/nginx/conf.d/
# 将静态资源复制到访问路径下
COPY ./dist/ /var/wwwroot

EXPOSE 80
```

```nginx
server {
  listen 80;
  server_name _;

  # 服务的配置
  location / {
    alias /var/wwwroot/;
    try_files $uri $uri/ /index.html =404;
  }
}
```

上述docker文件编写好后，分别进行build即可。

### 发现和注册服务

通过K8S注册一个服务后，如果是做了集群的，可以参考配置`Ingress`来启用服务发现，但我们目前仅使用到单集群单节点的方式，实际上是可以直接通过服务名来访问的。

比如我们部署了一个服务叫`username-func-test-k8s`，那么我们可以通过命令`curl username-func-test-k8s`来访问到服务本身。

回想我们要的效果是通过cookie来识别我们要访问的服务，所以只需要通过配置nginx解析cookie来做对应的反向代理即可。

这里值得说明的是nginx的反向代理是无法支持配置`search`的，也就是说没有办法通过变量的形式直接进行反向代理，比如：
```nginx
# 这样是不支持的
proxy_pass http://$SVC_NAME/
```

所以每次有新服务，我们都需要重新配置一条规则，看似需要有一个专门的服务来管理这件事，还记得我们CI/CD的最后一个阶段所做的事吗？

没错，它所访问的地址，正是我们的这个服务。可随意搭建一个http服务即可，核心逻辑如下：

```javascript
function updateNginxConfig (conf) {
  // 读取nginx模板文件
  const nginxTemplate = _.template(fs.readFileSync('nginx.template', { encoding: 'utf-8' }))
  // 读取服务列表
  const conf = getSvcList()
  // 根据服务列表生成新的nginx文件
  fs.writeFileSync('/etc/nginx/conf.d/default.conf', nginxTemplate({ services: conf }))
}

function getSvcList () {
  // 读取k8s服务列表，以josn输出方便使用
  const svcQuery = childProcess.spawnSync('kubectl', ['get', 'svc', '-o', 'json'], { encoding: 'utf-8' })

  if (svcQuery.stdout) {
    const svc = JSON.parse(svcQuery.stdout)
    // 排除一些基础服务，我们只要符合`${username}-${branch}-svc`规则的服务
    const gaiaSvc = svc.items.filter(item => /\w+-(\w+-)*svc/.test(item.metadata.name))
    return gaiaSvc.map(item => ({ name: item.metadata.name }))
  } else {
    throw new Error(svcQuery.stderr)
  }
}

// 这个接口用于返回符合规则的所有的服务列表
router.get('/', (req, res) => {
  try {
    return res.status(200).json({ list: getSvcList() })
  } catch (err) {
    return res.status(500).json({ msg: err })
  }
})

// CI中第四个阶段调用的方法
router.post('/', (req, res) => {
  // 生成并更新nginx配置文件
  updateNginxConfig()
  // 重启nginx服务
  reloadServer()

  return res.status(201).json()
})
```

最后我们来看看nginx的模板长什么样子：
```template
server {
  server_name  server.com;
  listen 80;

  proxy_next_upstream off;
  proxy_set_header    X-Real-IP           $remote_addr;
  proxy_set_header    X-Forwarded-For     $proxy_add_x_forwarded_for;
  proxy_set_header    Host                $host;
  proxy_http_version  1.1;
  proxy_set_header    Connection  "";
  proxy_set_header    Accept-Encoding "";
  charset utf-8;

  location / {
    set $svc '';

    # 获取cookie信息
    if ($http_cookie ~* "gaia_svc=(.+?)(?=;|$)") {
      set $svc $1;
    }

    # 如果没有指定环境，那么反向代理到一个默认的服务上去，避免出现50x错误
    if ($svc = '') {
        proxy_pass https://10.10.10.10;
    }

    # 根据k8s的服务，循环出判断题，最终根据cookie设置的服务名做反向代理
    <% _.forEach(services, function (svc) { %>
    if ($svc = <%= svc.name %>) {
        proxy_pass http://<%= svc.name %>;
    }
    <% }) %>

    # 注入环境选择器
    # 我们之前一直没有说明测试人员绑定了这个服务器后，应该怎么切换环境
    # 其实就是通过这个脚本来完成，我们可以注入一些页面元素，或者通过快捷键唤起环境选择器
    sub_filter_types text/html;
    sub_filter_once on;
    sub_filter '</body>' '<script src="https://server.com/env-selector.js" async></script></body>';
  }

  # 配置环境选择器的路径
  location /env-selector.js {
    alias /usr/share/nginx/html/js/env-selector.js;
  }

  // 配置环境选择器所需要用到的接口
  // 这里转发到本地的 get / 接口上，用于获取合法的环境列表
  location /__gaia_env__/ {
    proxy_pass http://127.0.0.1:3500/;
  }
}
```

环境选择器代码如下：

```javascript
import App from './App.svelte'

const DEFAULT_ENV = [{
  name: 'beta'
}]

document.addEventListener('keydown', function (evt) {
  var key = evt.which || evt.keyCode
  // 当用户按CTRL+SHIFT+Q即可唤起环境选择器
  if (evt.ctrlKey && evt.shiftKey && key == 81) {
    getEnvList(function (err, data) {
      if (err) {
        alert('获取环境列表失败')
      } else {
        const envList = data.list
        const app = new App({
          target: document.body,
          props: {
            env: DEFAULT_ENV.concat(envList)
          }
        })
        app.$set({ app })
      }
    })
  }
})

```

App.svelte
```svelte
<script>
  export let env
  export let app

  let selectEnv = ''
  const gaiaSvc = getCookie("gaia_svc") || "原Beta环境";
  const BETA_ENV = 'beta'

  function handleClick () {
    if (selectEnv) {
      selectEnv === BETA_ENV ? deleteCookie('gaia_svc') : setCookie("gaia_svc", selectEnv)
      window.location.reload()
    }
  }

  function handleClose () {
    app.$destroy()
  }
</script>

<div id="GaiaEnvSelector">
  <div class="gaiaCaption">
    <span class="gaiaEnvClose" on:click={handleClose}>X</span>
    环境选择
  </div>
  <div class="gaiaCurrentEnv">当前环境：<em>{gaiaSvc}</em></div>
  <select class="gaiaSvcSelect" bind:value="{selectEnv}">
    <option value="">请选择</option>
    {#each env as { name }, i}
    <option value="{ name }">{ name }</option>
    {/each}
  </select>
  <button class="switchEnvBtn" on:click={handleClick}>切换环境</button>
</div>
```

最终，所有步骤均已完成。