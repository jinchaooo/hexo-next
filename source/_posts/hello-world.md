---
title: Hello World
date: 2018-12-31 21:00:00
---
Welcome to [Hexo](https://hexo.io/)! This is your very first post. Check [documentation](https://hexo.io/docs/) for more info. If you get any problems when using Hexo, you can find the answer in [troubleshooting](https://hexo.io/docs/troubleshooting.html) or you can ask me on [GitHub](https://github.com/hexojs/hexo/issues).

## Quick Start

### Create a new post

``` bash
$ hexo new "My New Post"
```

More info: [Writing](https://hexo.io/docs/writing.html)

### Run server

``` bash
$ hexo server
```

More info: [Server](https://hexo.io/docs/server.html)

### Generate static files

``` bash
$ hexo generate # or hexo g
```

More info: [Generating](https://hexo.io/docs/generating.html)

### Deploy to remote sites

``` bash
$ hexo deploy # or hexo d
```

More info: [Deployment](https://hexo.io/docs/deployment.html)
<!--more-->

### How to create a clean Hexo + Next set up (tested on ubuntu 18.04)
1. install Node.js and npm
2. install hexo
``` bash
$ sudo npm install hexo-cli -g
```
3. initialize an empty github_pages blog
``` bash
$ hexo init name.github.io
$ cd name.github.io
$ npm install
```
4. create CNAME file with blog URL in source folder
5. copy blog posts into source/_posts folder
6. in _config.yml, change title, author, etc. change deploy section
``` bash
deploy:
  type: git
  repo: git@github.com:jinchaooo/jinchaooo.github.io.git
  branch: master
```
7. install git deployer 
``` bash
npm install hexo-deployer-git --save
```
8. to use the Next theme, clone it from github, and set theme section in _config.yml
``` bash
git clone https://github.com/theme-next/hexo-theme-next themes/next
```
9. Next theme supports Mathjax for math equation editing, to enable, change themes/next/_config.yml
``` bash
math:
  enable: true
  per_page: false
  engine: mathjax
```
10. install the renderer for mathjax [(link)](https://github.com/theme-next/hexo-theme-next/blob/master/docs/MATH.md)
``` bash
npm un hexo-renderer-marked --save
npm i hexo-renderer-kramed --save # or hexo-renderer-pandoc
```