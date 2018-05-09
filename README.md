## 新电脑部署博客步骤
1.在 Node.js 官网：https://nodejs.org/en/ 下载安装包 v6.10.3 LTS
 node -v
 npm -v

2.在 Git 官网：https://git-scm.com/ 下载安装包 Git-2.13.0-64-bit.exe

3.
npm install hexo-cli -g
hexo version

4.
hexo init bxm0927.github.io
cd bxm0927.github.io
npm install
hexo s

5.安装python 27 并且添加配置环境

6.
npm config set python "C:\Python27\python.exe"

7.
clone https://github.com/KaelPu/blog.git 
并且覆盖 kaelpu.github.io文件

8.
hexo clean
hexo g
hexo s --debug
hexo d 上传