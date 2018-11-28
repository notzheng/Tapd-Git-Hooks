# Tapd Git Hooks

## 说明

Tapd 平台的的源码关联功能目前只支持 Github 、 Gitlab 与 Tgit 的源码关联。

但是也有很多公司在使用 Gogs、Gitee、Coding 这些 Git 平台，无法使用源码关联功能。通过测试发现，这些平台在触发 webhook 时，向目标网址提交的 payload 数据结构是相似的，核心内容是相同的。区别仅仅在于请求头的字段名称。

例如，Gogs 与 Github 请求头的差异：

|       Gogs       |      Github       |                 内容                 |
| :--------------: | :---------------: | :----------------------------------: |
| X-Gogs-Delivery  | X-GitHub-Delivery | 884d89aa-f1fb-11e8-869b-9bf6f51d58b5 |
|   X-Gogs-Event   |  X-GitHub-Event   |                 push                 |
| X-Gogs-Signature |  X-Hub-Signature  |        当设置了 Secret Token         |
|    User-Agent    |         /         |       GitHub-Hookshot/7423407        |

所以，我们可以在自己的服务器上假设一个中间层，通过反向代理的方式改变（添加）请求头之后，将这些不被支持的平台“伪装”成 Github 或 Gitlab 等被支持的平台转发到 tapd 的 webhook 地址即可。

比如，tapd 提供的 Hook 地址是：

```
https://hook.tapd.cn/64911721/f002ca6d79322162da8622af37087412
```

我们的中间层域名是 `tapdhook.example.com`

那么，我们在 Git 平台的 Webhook 中填入

```
https://tapdhook.example.com/64911721/f002ca6d79322162da8622af37087412
```

即可。


这个方案经过我们的测试是可行的，所以决定将它分享出来。但是这种方案实在是不得已而为之，也希望 Tapd 平台的程序员们能够在一些加入对这些平台的支持。

实现这个转发，有两种方式，一种是直接使用 Nginx 反向代理；另一种是在应用层来做，后端到请求进行处理，之后再 POST 到 Tapd 的 Hook 网址，这也可以解决掉 Secret Token 的问题。

#### Secret Token 问题


测试过程中我们发现，不同平台的 Signature 算法是不同的，所以部分平台可能无法**简单**的使用 Secret Token 功能。比如我们使用的 Gogs ，它的 Signature 使用的是 sha256 算法，与 Github 的 sha1 不同。这个通过 Nginx 反向代理是无法解决的。两种方式：

1. 在中间层的 webhook URL 中带上 Secret Token ，但这样子也就失去意义；改进一点的话可以约定一种非对称密加密方式。
2. 开发一个完整的后端程序。可以将 Secret Token 存储到数据库中，这种方式自由度就比较高了。

但是中间层一般部署在自己的内网服务器上，且 URL 地址的不公开本身也是一种保护方案。考虑到 Gogs 平台以及人手问题，我们现在仅仅是用了 Nginx 反向代理功能，没有解决 Secret Token 问题，请大家自行取舍。


| 反向代理支持 Secret Token | 需后端实现 Secret Token |
| ------------------------- | ----------------------- |
| Gitee                     | Gogs                    |

#### 贡献

Git 平台有非常多，由于精力有限，我没有测试很多，欢迎大家通过 PR 的方式参与贡献，在此表示感谢！

如果哪家公司可以实现一个后端解决方案，欢迎开源出来让大家学习使用。




## 配置示例

这里都以 Nginx 为例，在 conf 文件夹中我们给出了示例配置文件。

### Gogs

Gogs 的请求头格式与 Github 相同，所以需要在项目中使用 Github 源码关联功能。

对应关系：

|       Gogs       |      Github       |                 内容                 |
| :--------------: | :---------------: | :----------------------------------: |
| X-Gogs-Delivery  | X-GitHub-Delivery | 884d89aa-f1fb-11e8-869b-9bf6f51d58b5 |
|   X-Gogs-Event   |  X-GitHub-Event   |                 push                 |
| X-Gogs-Signature |  X-Hub-Signature  |        当设置了 Secret Token         |
|    User-Agent    |         /         |       GitHub-Hookshot/7423407        |

但是 Gogs 的 Signature 是 sha256 算法，Github 是 sha1 算法，所以不能直接更改。

因此，如果要设置 Secret Token，还是需要应用层。这一点在前言中有说到。

```ini
  # 匹配 tapd webhooks 的路径
  location ~ ^/(\d+)/([a-z0-9]+) {
    # 将 Gogs 的请求头设置成 Github 的请求头
    proxy_set_header X-GitHub-Delivery $http_X_Gogs_Delivery ;
    proxy_set_header X-GitHub-Event $http_X_Gogs_Event ;
    proxy_set_header X-Hub-Signature $http_X_Gogs_Signature ;
    proxy_set_header Expect "$http_X_Gogs_Signature" ;
    proxy_set_header User-Agent "GitHub-Hookshot/7423407" ;
    # 反向代理到 tapd 
    proxy_pass https://hook.tapd.cn ;
  }
```

### 

###Gitee

Gitee 的请求头格式与 Gitlab 相同，所以需要在项目中使用 Gitlab 源码关联功能。

对应关系：

|     Gitlab     |      Gitee       |                 内容              |
| :------------: | :--------------: | :--------------------------------:|
| X-Gitlab-Event |  X-Gitee-Event   |            Push Hook              |
| X-Gitlab-Token |  X-Gitee-Token   |        Secret Token       |

幸运的是，Gitee 与 Gitlab 均没有对 Secret Token 进行加密，因此可以使用该功能。

```ini
  # 匹配 tapd webhooks 的路径
  location ~ ^/(\d+)/([a-z0-9]+) {
    # 将 Gitee 的请求头设置成 Gitlab 的请求头
    proxy_set_header X-Gitlab-Event $http_X_Gitee_Event ;
    proxy_set_header X-Gitlab-Token $http_X_Gitee_Token ;
    # 反向代理到 tapd 
    proxy_pass https://hook.tapd.cn ;
  }
```

# License

[![WTFPL](http://www.wtfpl.net/wp-content/uploads/2012/12/wtfpl-badge-1.png)](http://www.wtfpl.net/)

