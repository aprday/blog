---
title: 使用css构造等比例布局
date: 2017-08-03 11:46:13
tags: Javascript
---

### css等比例图片

##### 神奇的margin-top:

http://www.songxuemeng.com/vue-demo/demo/css/margin-top.html

margin-top是根据父元素的宽度来进行计算的

##### 等比例less
    .aspect-ratio(@width, @height) {
    position: relative;
    &:before {
     display: block;
     content: "";
     width: 100%;
     padding-top: (@height / @width) * 100%;
    }
    > .content {
     position: absolute;
     top: 0;
     left: 0;
     right: 0;
     bottom: 0;
    }
    }
##### 制作等比例图片

http://www.songxuemeng.com/vue-demo/demo/css/ratio-image.html

    .aspect-ratio(1, 1);
            .content{
              background-repeat: no-repeat;
              text-align: center;
            }

#### 使用margin-top保持间隙

http://www.songxuemeng.com/vue-demo/demo/css/space-image.html
