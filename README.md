# lightnine.github.io

个人博客

本地开发只需要hexo分支，在hexo上添加博客。编写完成后执行hexo clean && hexo g && hexo d, 即可部署到github pages上。

## hexo 常用命令
- hexo n "我的博客" == hexo new "我的博客" #新建文章
- hexo p == hexo publish
- hexo g == hexo generate #生成
- hexo s == hexo server #启动服务本地预览
- hexo d == hexo deploy #部署
- hexo clean #清除缓存 网页正常情况下可以忽略此条命令

- hexo server #Hexo 会监视文件变动并自动更新，您无须重启服务器。
- hexo server -s #静态模式
- hexo server -p 5000 #更改端口  --debug 显示日志信息
- hexo server -i 192.168.1.1 #自定义 IP


参考：[hexo命令使用](https://github.com/pengwenwu/skill-tree/blob/master/Hexo/hexo%20+%20github%E6%90%AD%E5%BB%BA%E4%B8%AA%E4%BA%BA%E5%8D%9A%E5%AE%A2%E6%95%99%E7%A8%8B.md)