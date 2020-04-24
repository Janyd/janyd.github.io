# 基于Hugo搭建博客


# 基于Hugo搭建博客
第一次写博客经验不足呀，请各位多多担待

## 搭建环境
- 操作系统：MacOS 10.15.3
- 工具：Git、VSCode
- 框架：[Hugo](https://github.com/gohugoio/hugo)、Github Pages
- 主题：[LoveIt](https://github.com/dillonzq/LoveIt) (感谢dillonzq提供的主题

# 具体步骤

## 安装Git
MacOS平台下使用git提示需要安装Xcode，如果是Windows平台直接到[Git官网](https://git-scm.com/)下载即可，安装过程也很简单。

## 安装Hugo
### MacOS平台
- 可以使用 `brew install hugo`
- [Hugo Github Releases](https://github.com/gohugoio/hugo/releases)下载相应的压缩包，解压出hugo可执行文件到`/usr/local/bin`下，就可以在命令行里调用`hugo`

### Windows平台
- 使用`Chocolatey`包管理安装，`choco install hugo`
- 同MacOS平台一样也可以到[Hugo Github Releases](https://github.com/gohugoio/hugo/releases)下载解压，并加入到环境变量下即可在命令行工具里调用`hugo`命令

按照以上步骤就可以在命令行里输入 `hugo version` 就会得出以下版本信息，那就可以使用hugo进行创建博客了
![](https://cdn.jsdelivr.net/gh/Janyd/blog-images/imgs/20200423224427.png)

## 创建站点
在你想要创建目录下输入
```Bash
hugo new site 站点名称
```
或者在任意目录下输入
```Bash
hugo new site 绝对路径/站点名称
```
就会输出以下提示，在此我创建的站点名称为`demo`  
![](https://cdn.jsdelivr.net/gh/Janyd/blog-images/imgs/20200423225755.png)  

## 添加主题
以当前主题为例  
进入到你新建的站点下的`themes`，克隆[LoveIt](https://github.com/dillonzq/LoveIt):
```Bash
git clone -b master https://github.com/dillonzq/LoveIt.git themes/LoveIt
```
或者，初始化你的项目目录为Git仓库，并且把主题仓库作为子模块:
```Bash
git init
git submodule -b master add https://github.com/dillonzq/LoveIt.git themes/LoveIt
```
关于主题的基础配置可到主题作者博客[主题文档 - 基本概念](https://hugoloveit.com/zh-cn/theme-documentation-basics/)进行配置  

## 添加第一篇文章
在站点目录下输入以下命令即可创建文章
```Bash
hugo new posts/first-post.md
```
文件内容会有title、date与draft，draft表示当前文章是否为草稿，默认是true，发布时不会将草稿发布

## 本地启动博客
在站点目录下输入以下命令即可启动，打开命令显示的网址就可以欣赏你写的博客啦！(-D选项是为了能够构建草稿的文章)
```Bash
hugo server -D
```

## 发布博客
在站点目录输入以下命令，hugo会将markdown文章转为html静态文件放在了`public`目录下
```Bash
hugo
```
因为可通过多种方式放到网站阅读，例如(举例部分)：
- nginx代理
- Github Pages
- Gitee Pages  

会在文章放出网上别人做如何发布博客的参考链接，大家可以前往看看

# 总结
搭建过程还是比较容易的，之前尝试过Hexo也算比较好的，大家可以去作对比选择一款适合自己的框架，之所以选择Hugo因为官网的一句`The world’s fastest framework for building websites`。接下来会分享一些源码解析，本人还是个新人，写的不好请大家指出。

## 参考链接
- [Hugo官网](https://gohugo.io/)
- [LoveIt主题博客](https://hugoloveit.com/zh-cn/theme-documentation-basics/#site-configuration)
- [Hugo + Github Pages 搭建个人博客](https://www.cnblogs.com/stevexu/p/10779375.html)
- [在macOS上使用GitHub Pages+Hexo搭建个人博客](https://jeromememory.github.io/2019/08/08/%E5%9C%A8macOS%E4%B8%8A%E4%BD%BF%E7%94%A8GitHub-Pages-Hexo%E6%90%AD%E5%BB%BA%E4%B8%AA%E4%BA%BA%E5%8D%9A%E5%AE%A2.html)
- [Hugo 常用命令详解](https://blog.csdn.net/weixin_33769125/article/details/93111054)
