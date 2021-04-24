---
title: manimgl install
date: 2021-04-24 21:59:33
tags: manimgl
---

# manimgl安装

准备用`manimgl`渲染一些数学释义，配置tex的过程折腾了一番，记录一下备忘。

```bash
pip install tex
sudo apt-get install texlive-latex-base dvipng texlive-latex-extra cm-super xzdec
# setup tlmgr
tlmgr init-usertree
tlmgr option repository ftp://tug.org/historic/systems/texlive/2017/tlnet-final
# update tlmgr
#wget http://mirror.ctan.org/systems/texlive/tlnet/update-tlmgr-latest.sh
#sh update-tlmgr-latest.sh -- --upgrade
# install physics.sty
sudo $(which tlmgr) install physics
```

