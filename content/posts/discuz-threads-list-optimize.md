---
title: Discuz!顶置贴、帖子列表优化建议
date: 2013-07-22T00:00:00+08:00
draft: false
tags: ["discuz", "优化", "置顶贴", "帖子列表"]
slug: discuz-threads-list-optimize
---

## 顶置贴的存储

表 pre_forum_thread 字段 displayorder

    4    多版块顶置。Disczu!7.2引入的一个特性，操作入口在“论坛”－“版块/群组顶置”
    3    3级置顶、全局顶置
    2    2级置顶、分区顶置
    1    1级置顶、版块顶置
    0    正常
    -1    回收站
    -2    审核中
    -3    审核忽略
    -4    草稿

- 顶置2、3类的tid会被保存在pre_common_syscache的cname="globalstick"记录中
- 顶置2 `$_G['cache']['globalstick']['categories'][分区ID]`
- 顶置3 $_G['cache']['globalstick']['global']['tids']
- 多版块顶置数据（4）被保存在pre_common_syscache的cname="forumstick"记录中
- $_G['cache']['forumstick'][版块ID]

## 帖子列表页对顶置贴的显示逻辑。

Discuz!通过下面的SQL查询检索上面3种置顶帖子数据。

第一页

```
SELECT * FROM pre_forum_thread WHERE `tid` IN('3','13','12','5') AND `displayorder` IN('2','3','4') ORDER BY displayorder DESC, lastpost DESC LIMIT 20
Using where; Using filesort
```

第二页

```
SELECT * FROM pre_forum_thread WHERE `tid` IN('3','13','12','5') AND `displayorder` IN('2','3','4') ORDER BY displayorder DESC, lastpost DESC LIMIT 20, 20
Using where; Using filesort
```

可以看到，即使第二页没有需要显示的顶置贴，但仍然会执行查询，这里可以做个小小的优化，计算一下IN（）表达式中的tid的数量。
然后是排序部分，如果条件允许的话，建议通过PHP脚本来排序。是否使用这一项优化，主要受版块中顶置贴的数量影响。如果顶置贴数量较多，必须要通过分页来显示，就需要在数据库中排序后，才能确定在指定页面要显示的顶置贴是哪些。不过按通常的情况来看，版块中顶置贴的数量都还是比较少，很少会出现顶置贴翻页的情况。

本版顶置贴与普通贴通过类似下面的SQL语句检索。

```
SELECT * FROM pre_forum_thread WHERE `fid`='2' AND `displayorder` IN('0','1') ORDER BY displayorder DESC, lastpost DESC LIMIT 20
```

建议本版顶置的帖子也单独进行维护，类似置顶2，3，4的情形。

基于置顶贴在站点所有帖子中的比重很少这个观点，displayorder字段索引的“0”值将覆盖占据绝大多数。理想情况下，对非0值的索引才是有意义的。为了实现这个目标，我们需要把displayorder为负数的数据分离。

理论上，将这些不显示在帖子列表页中的数据从 pre_forum_thread 表中分离使得表中剩下的数据都是要正常显示的内容，可以减少表的大小，显示帖子列表页也不再需要依赖displayorder列组织查询。

一种方式，平行拆分。拆分出一个结构和 pre_forum_thread 数据结构完成一样的表格，存储 displayorder 小于0的数据。需要注意的是，可能需要一个 tid 生成器，因为草稿和待审核的帖子tid将不能通过 pre_forum_thread 表的自增方式生成。

第二种方式，借鉴linux文件系统模型（inode+文件数据实体），这里的帖子数据也可以这样来组织：帖子数据（标题、内容、附件、作者等）是数据实体，帖子在系统中的属性（顶置、精华、草稿等）是inode的内容。即把帖子实体独立成表，它是与状态无关的内容；逻辑属性相关的数据（不包含帖子实体数据）组织成另外的表，是实现各种条件查询、过滤等执行的数据表，依系统数据的规模而定，比如，正常显示的帖子（displayorder > 0）一个表，不显示的帖子（displayorder < 0）一个表，又或者displayorder非零的组织成一个表。从这些“属性相关”表中找出要显示的帖子tid后，去帖子实体表中取数据显示。这样pre_forum_thread表会被拆分成两个或者三个表，独立出来的类inode表应该是非常小的固定长度的表，查询速度是很快的，索引也可以排除掉大面积覆盖的键值。当然，实际的运行效率还需要在生产数据中进行验证。

### 与“全局置顶”相关的选项：

界面－界面设置－主题列表页－启用全局置顶。
论坛－版块管理－扩展设置－显示全局置顶和分类置顶的主题
用户－管理组－主题管理权限－允许置顶主题
