---
title: java里的坑
date: 2019-01-22 11:21:38
categories: Java
tags:
	- java
---

Comparator

在jdk1.6可以返回1或0，在jdk1.7必须返回一对相反数，像下面的Comparator是不工作的
```java
private class SortByUseCount implements Comparator<ItemInfo> {

    @Override
    public int compare(ItemInfo o1, ItemInfo o2) {
        return o1.useCount < o2.useCount ? 0 : 1;
    }
}
```