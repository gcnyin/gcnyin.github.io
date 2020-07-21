---
layout: post
title: "评论系统的表设计"
date: 2020-07-21 21:00:00 +0800
---

## 前言

主流的评论系统功能有三种：

- 一级评论。例如朋友圈。
- 最多两级评论。例如微博。
- 无限级评论。例如网易新闻。

## 第一种

只需要一张表。

```
id 评论自己的ID
discuss_id 对应帖子的ID
author_id 作者ID
reply_to 被回复者的ID（可空）
content 回复的内容
```

## 第二种

```
id 评论自己的ID
discuss_id 对应帖子的ID
author_id 作者ID
replied_comment_id 被回复的评论ID
level 层级（1或者2）
content 回复的内容
```

## 第三种

这种相对比较复杂，需要支持递归查询，在表中必须有`parent_comment_id`字段。

用一个简单的例子说明递归查询的用法。创建表并插入数据：

```sql
CREATE TABLE IF NOT EXISTS "comments" (
    id VARCHAR(36) PRIMARY KEY NOT NULL,
    parent_id VARCHAR(36),
    "content" INT NOT NULL
);

INSERT INTO "comments" VALUES ('1', NULL, random() * 10000 + 1);
INSERT INTO "comments" VALUES ('2', '1', random() * 10000 + 1);
INSERT INTO "comments" VALUES ('3', '2', random() * 10000 + 1);
INSERT INTO "comments" VALUES ('4', '1', random() * 10000 + 1);
```

递归查询需要使用`with`和`recursive`关键字，详情见文末参考资料。

```sql
WITH RECURSIVE comments_graph ( id, content, parent_id, depth) AS (
    SELECT id, content, parent_id, 1
    FROM "comments"
    WHERE "comments".id = '1'
    UNION ALL
    SELECT c.id, c."content", c.parent_id, depth + 1
    FROM "comments" c, comments_graph graph
    WHERE c.parent_id = graph.id
)
SELECT * FROM comments_graph;
```

结果：

```
id,content,parent_id,depth
1,2397,,1
2,3254,1,2
4,380,1,2
3,9715,2,3
```

## 参考资料

- https://www.postgresql.org/docs/current/queries-with.html
- https://cra.mr/2010/05/30/scaling-threaded-comments-on-django-at-disqus/