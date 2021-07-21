---
title:  "PDF渲染"
date:   2021-07-21
categories: Project
---

![render flow](/images/render_flow.png)

## 渲染请求

kRequestTypeUrgent：当前需要显示的页面，优先级最高。
kRequestTypeThumbnail：单页模式、对开模式下缩略图，即两种模式下，拖动滚动条时预览展示。
kRequestTypePreRender：预先渲染的页面，即预先渲染前一页、后一页，对开模式下预先渲染前两页、后两页。

每种渲染请求有一个request manager，每个request manager管理固定大小（MAX_REQUEST_COUNT）的请求数目。采用多个request manager可以防止kRequestTypeUrgent请求被其他请求覆盖，添加请求采用先进先出的策略，即请求队列满时，移除最旧的请求。

主线程处理用户鼠标、键盘消息触发导航操作，并根据坐标信息添加渲染请求（只请求页面显示部分），唤醒渲染线程进行渲染，并Invalidate窗口。

主线程收到WM_PAINT消息后，根据坐标信息到内存缓存中查找页面显示的部分。如果有匹配的缓存则将内容BitBlt到HDC上，并将页面unfinished part添加到请求；如果没有匹配的缓存，则查找能够通过拉伸、压缩之后匹配的缓存，并将之StretchBlt到HDC上。

## 渲染操作

页面渲染在渲染线程进行，通过调用PDF渲染库获得页面指定范围的内容。

页面渲染前，根据主线程设置的页面信息（原子变量），过滤掉不需要的渲染；
页面渲染时，采用优先kRequestTypeUrgent类型策略，单独request manager内部请求采用后进先出，即优先渲染后添加的请求；
页面渲染后，将页面内容添加到内存缓存，内存缓存策略：当页面大小小于指定阈值时，则缓存完整页面，页面中unfinished part填充背景色。当页面大小超过指定阈值时，则只缓存请求部分页面，防止内存分配失败、内存耗尽等情况出现。添加完缓存后，通知主线程将内容上屏（Invalidate窗口）。

## 缓存

缓存通过页码进行分组，从而减小锁的粒度。

硬盘缓存：渲染线程获取到渲染请求后，首先在硬盘缓存中查找，如果存在匹配entry则从硬盘加载相应缓存；如果不存在匹配entry则发起渲染操作。

内存缓存: 保留当前页、前后两页（对开模式有些不同）内容，其他页面缓存从内存中移除。移除过程中判断页面是否已经渲染完整（即finished part等于完整页面），完整的页面则放入硬盘缓存。
