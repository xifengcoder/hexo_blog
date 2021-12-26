---
title: hexo个性化配置
urlname: hexo_config
date: 2021-12-26 21:23:56
tags: hexo
categories: hexo
description:
top: true
---

在next主题配置过程中，做了一些个性化修改，包括图片模式、博文置顶、背景修改、主题颜色、不蒜子统计显示、修改内容宽度以及文章分割线长度和间距修改。

<!-- more -->

#### 首页文章上下间距以及文章分割线长度修改

修改分割线宽度：

themes\hexo-theme-next-8.8.0\source\css\_common\components\post\post-footer.styl

```css
.post-eof {
  background: $grey-light;
  height: 1px;
  margin: $post-eof-margin-top auto $post-eof-margin-bottom;
  width: 75%; # 分割线宽度

  .post-block:last-of-type & {
    display: none;
  }
}

```

修改上下间距：

themes\hexo-theme-next-8.8.0\source\css\_variable\base.styl

```css
$post-eof-margin-top          = 40px; //  or 160px for more white space;
$post-eof-margin-bottom       = 30px; //  or 120px for less white space;
```

有一篇介绍比较详细的文章，解决了我不少问题，特表感谢：

 Next主题的安装、优化、修改（http://sixiwenwu.com/88/）
