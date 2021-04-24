---
title: Hello World
date: 2020-06-29 19:29:44
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
$ hexo generate
```

More info: [Generating](https://hexo.io/docs/generating.html)

### Deploy to remote sites

``` bash
$ hexo deploy
```

More info: [Deployment](https://hexo.io/docs/one-command-deployment.html)

## Mermaid

installation:

```bash
npm install hexo-mermaid-diagrams
```

usage:

```markdown
{% mermaid type %}
{% endmermaid %}
```

e.g.

```markdown
{% mermaid %}
graph TD;
    A-->B;
    A-->C;
    B-->D;
    C-->D;
{% endmermaid %}
```

{% mermaid %}
graph TD;
    A-->B;
    A-->C;
    B-->D;
    C-->D;
{% endmermaid %}

## Math

默认render会把latex公式中的`_`渲染成HTML中的`<i>`, 需要更换render。

```bash
npm install hexo-math --save
cd PATH_TO_YOUR_HEXO_BLOG
hexo math install
# NOW, add plugins: hexo-math into the _config.yml
npm uninstall hexo-renderer-marked --save
npm install hexo-renderer-kramed --save
```

注意， `kramed`的inline公式需要改变写法，增加单行代码引用包围， 例如：

```
`$a_t$`
```

