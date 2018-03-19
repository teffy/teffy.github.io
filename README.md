# teffy.github.io
My blog powered with HEXO.

```
git clone git@github.com:teffy/teffy.github.io.git
# submodule
git submoudle init
git submodule update
# hexo
npm install hexo --save
# include-markdown
npm install hexo-include-markdown --save
# sitemap
npm install hexo-generator-sitemap --save-dev
npm install hexo-generator-baidu-sitemap --save-dev
# baidu url submit
npm install hexo-baidu-url-submit --save
# wordcount
npm i hexo-wordcount --save
```

使用markdown include插件
https://github.com/tea3/hexo-include-markdown

http://theme-next.iissnan.com/third-party-services.html
https://pengbinlee.github.io/Hexo-NexT-%E4%B8%BB%E9%A2%98%E7%9A%84-SEO%E4%BC%98%E5%8C%96/
https://github.com/iissnan/hexo-theme-next/wiki/%E6%98%BE%E7%A4%BA-feed-%E9%93%BE%E6%8E%A5
http://theme-next.iissnan.com/faqs.html#custom-content-width
https://www.jianshu.com/p/0d54a590b81a
https://www.jianshu.com/p/ab44b916a8b6
http://www.wuxubj.cn/2016/08/Hexo-nexT-build-personal-blog/


这篇验证过的：
https://www.jianshu.com/p/9be9b4786f97
seo
http://blog.csdn.net/sunshine940326/article/details/70936988
https://www.jianshu.com/p/9be9b4786f97
https://www.jianshu.com/p/86557c34b671

[toc]
https://imys.net/20150514/hexo-toc.html


版权
http://www.crocutax.com/2017/05/20/Hexo%E6%8C%81%E7%BB%AD%E4%BC%98%E5%8C%96-%E5%9C%A8%E6%96%87%E7%AB%A0%E5%B0%BE%E9%83%A8%E6%B7%BB%E5%8A%A0%E7%89%88%E6%9D%83%E5%A3%B0%E6%98%8E%E4%BF%A1%E6%81%AF/

http://www.bijishequ.com/detail/392652?p=56-51-64
http://www.itfanr.cc/2017/12/06/hexo-blog-optimization/

fancybox-thumbs.css ENOSPC:
```
先测试npm dedupe
不行再试这个
echo fs.inotify.max_user_watches=524288 | sudo tee -a /etc/sysctl.conf && sudo sysctl -p
```
http://www.sail.name/2017/10/31/ENOSPC-Error(Linux)-in-hexo/


```
Plugin loaded: hexo-baidu-url-submit
Plugin loaded: hexo-generator-archive
Plugin loaded: hexo-deployer-git
Plugin loaded: hexo-generator-category
Plugin loaded: hexo-generator-index
Plugin loaded: hexo-generator-searchdb
Plugin loaded: hexo-generator-tag
Plugin loaded: hexo-include-markdown
Plugin loaded: hexo-renderer-marked
Plugin loaded: hexo-renderer-ejs
Plugin loaded: hexo-renderer-stylus
Plugin loaded: hexo-server
Plugin loaded: hexo-wordcount
Plugin loaded: hexo-generator-baidu-sitemap
Plugin loaded: hexo-generator-sitemap
Script loaded: themes/hexo-theme-next/scripts/merge-configs.js
Script loaded: themes/hexo-theme-next/scripts/tags/button.js
Script loaded: themes/hexo-theme-next/scripts/tags/center-quote.js
Script loaded: themes/hexo-theme-next/scripts/merge.js
Script loaded: themes/hexo-theme-next/scripts/tags/exturl.js
Script loaded: themes/hexo-theme-next/scripts/tags/full-image.js
Script loaded: themes/hexo-theme-next/scripts/tags/group-pictures.js
Script loaded: themes/hexo-theme-next/scripts/tags/label.js
Script loaded: themes/hexo-theme-next/scripts/tags/lazy-image.js
Script loaded: themes/hexo-theme-next/scripts/tags/note.js
Script loaded: themes/hexo-theme-next/scripts/tags/tabs.js
```