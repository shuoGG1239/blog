---
title: 博客主题icarus样式魔改
date: 2024/11/27
categories: 
- 工具
tags:
- hexo
---

#### 简介
* 这几天把博客主题从next换成了icarus, 官方默认样式不大满意, 我做了些调整, 这里记下调整过程


#### 预览(前者官方, 后者魔改)
![easyscreenshot20241128211047.jpg](https://s2.loli.net/2024/11/28/hYTn9s7V1vFQP6G.jpg)
![easyscreenshot20241128211128.jpg](https://s2.loli.net/2024/11/28/GOtbCjoenPgylE6.jpg)


#### 小组件调整
* 自带的小组件太杂了, 这里我只保留了总览, 分类, 标签, 近期文章
* 修改`_config.icarus.yml`的`widgets`, 仅保留`type`为`profile`, `categories`, `recent_posts`, `tags`这4个, 其他删掉或注释掉即可


#### 代码样式调整
* 修改`_config.icarus.yml`的`article/highlight/theme`, 默认是用`atom-one-light`, 我改用`vs`即vscode的样式


#### 标题样式调整
* 原大标题字体没加粗处理, 跟文章content看起来不好区分, 所以做了加粗处理
* 修改`include/style/article.styl`文件, 在h1到h5项分别加上`font-weight: 600`, 如下:
```styl
    &.article
        .article-meta, .article-tags
            color: $text-light

        .article-meta
            overflow-x: auto
            margin-bottom: .5rem

        .article-more
            @extend .button.is-light

        .content
            word-wrap: break-word
            font-size: $article-font-size

            h1
                font-weight: 600
                font-size: 1.75em

            h2
                font-weight: 600
                font-size: 1.5em

            h3
                font-weight: 600
                font-size: 1.25em

            h4
                font-weight: 600
                font-size: 1.125em

            h5
                font-weight: 600
                font-size: 1em
```


#### 文章宽度和留空区宽度调整
* 默认的文章宽度太小, 两边的留空区太大, 文章内部就显得拥挤
1. 调整`include/style/base.styl`的基础变量`$gap`, 由`64px`改为`32px`
```styl
$gap ?= 32px
$tablet ?= 769px
$desktop ?= 1088px
$widescreen ?= 1280px
$fullhd ?= 1472px
```

2. 调整`include/style/responsive.styl`里面containter的width, container其实就是文章容器, 这里把默认值`2 * $gap`改成`1 * $gap`
```styl
+widescreen()
    .is-1-column .container, .is-2-column .container
        max-width: $widescreen  - 1 * $gap
        width: $widescreen  - 1 * $gap

+fullhd()
    .is-2-column .container
        max-width: $fullhd  - 1 * $gap
        width: $fullhd  - 1 * $gap

    .is-1-column .container
        max-width: $desktop - 1 * $gap
        width: $desktop - 1 * $gap
```

3. 调整`layout/common/widgets.jsx`的`getColumnSizeClass`, 将`case 2`的`is-4-tablet is-4-desktop is-4-widescreen`替换为`is-2-tablet is-2-desktop is-2-widescreen`
```styl
function getColumnSizeClass(columnCount) {
    switch (columnCount) {
        case 2:
            return 'is-2-tablet is-2-desktop is-2-widescreen';
        case 3:
            return 'is-3-tablet is-3-desktop is-3-widescreen';
    }
    return '';
}
```

4. 调整`layout/layout.jsx`, 将`'is-8-tablet is-8-desktop is-8-widescreen': columnCount === 2`替换为`'is-8-tablet is-9-desktop is-9-widescreen': columnCount === 2`
```styl
    <div class={classname({
        column: true,
        'order-2': true,
        'column-main': true,
        'is-12': columnCount === 1,
        'is-8-tablet is-9-desktop is-9-widescreen': columnCount === 2,
        'is-8-tablet is-9-desktop is-6-widescreen': columnCount === 3
    })} dangerouslySetInnerHTML={{ __html: body }}></div>
```

#### 侧边栏的profile
* 由于侧边栏被我调窄了, 头像显得太大, 所以这里调下`layout/widget/profile.jsx`的`figure`, 将`128x128`替换为`64x64`
```html
    <div>
        <figure class="image is-64x64 mx-auto mb-2">
            <img class={'avatar' + (avatarRounded ? ' is-rounded' : '')} src={avatar} alt={author} />
        </figure>
        {author ? <p class="title is-size-4 is-block" style={{'line-height': 'inherit'}}>{author}</p> : null}
        {authorTitle ? <p class="is-size-6 is-block">{authorTitle}</p> : null}
        {location ? <p class="is-size-6 is-flex justify-content-center">
            <i class="fas fa-map-marker-alt mr-1"></i>
            <span>{location}</span>
        </p> : null}
    </div>
```