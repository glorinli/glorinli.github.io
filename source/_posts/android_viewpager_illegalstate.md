---
title: 【Android】ViewPager 出现IllegalStateException
date: 2018-08-22
---
# 问题
使用ViewPager，动态加载页面，尽管修改了数据后，已经在主线程中调用了notifyDataSetChanged，还是出现以下错误

``` java
java.lang.IllegalStateException: The application's PagerAdapter changed the adapter's contents without calling PagerAdapter#notifyDataSetChanged! Expected adapter item count: 6, found: 3 Pager id: ...
```

# 原因
在调用 mAdapter.notifyDataSetChanged 之前，先调用了 viewPager.setOffscreenPageLimit 方法
此方法会调用 ViewPager 的 populate(); 方法，就是在这里出现了问题：Adapter.getCount() 已经改变，而 mExpectedAdapterCount 由于还没有调用 notifyDataSetChanged 所以没有更新。所以抛出异常。

# 解决方法
先调用 notifyDataSetChanged, 再调用 viewPager.setOffscreenPageLimit 即可。
