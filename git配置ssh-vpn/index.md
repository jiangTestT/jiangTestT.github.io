# Git 配置 SSH VPN


在软件开发过程中，频繁与 GitHub 交互是常态。然而，这些工具的连接不稳定往往让人困扰。

## Git 代理设置

Git 与 GitHub 仓库的连接主要通过两种协议进行：HTTPS 和 SSH。

首先，让我们来设置 Git 的 HTTPS 代理。

```sh
git config --global http.proxy 'socks5://127.0.0.1:7890'
git config --global https.proxy 'socks5://127.0.0.1:7890'
```

如果将来不需要代理，可以通过以下命令撤销之前的代理配置：

```sh
git config --global --unset http.proxy
git config --global --unset https.proxy
```

对于使用 SSH 协议的用户，代理的设置稍有不同。首先，打开或创建 GitHub 的 SSH 配置文件 config。这个文件位于 C:\Users\<user name>\.ssh。如果没有找到这个文件，请手动创建。然后，在该文件中添加以下内容来设置代理：

```shell
# 全局
ProxyCommand connect -S 127.0.0.1:7890 %h %p

# 只为特定域名设定
Host github.com
ProxyCommand connect -S 127.0.0.1:7890 %h %p
```

最后保存 config 文件，SSH 代理即完成设置。

通过这些步骤，无论是通过 HTTPS 还是 SSH 连接 GitHub，你都能享受到更加流畅和稳定的体验。
