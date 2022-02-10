---
title: UITableView-Invalid-update
date: 2022-02-10 17:32:11
categories:
- iOS
tags:
- UITableView Error
keywords:
- UITableView,Invalid,update
description:
images:
---
```
控制台错误日志：
Invalid update: invalid number of rows in section 0. The number of
rows contained in an existing section after the update (3) must be
equal to the number of rows contained in that section before the
update (2), plus or minus the number of rows inserted or deleted from
that section (0 inserted, 0 deleted) and plus or minus the number of
rows moved into or out of that section (0 moved in, 0 moved out).

无效更新：节0中的行数无效。行数
更新（3）后现有节中包含的行必须是
等于该节中包含的行数，在
更新（2），加上或减去从中插入或删除的行数
该部分（插入0，删除0）加上或减去
移入或移出该节的行（0移入，0移出）。
```
<!-- more -->
### 出现情况
```
<!--UITableView 处于 reload 中-->
tableView.beginUpdates()
tableView.endUpdates()
```

### 出现原因
当 UITableView section 中的数量有变化时并且此时 UITableView 正处于 reload 状态，此时未 reload 完的 UITableView 的section 中 cell数与reload 完的数量不一致，这个时候对于 UITableView 中 cell 的操作（如：增、删）就会导致该错误。

### 解决方案
1. 在beginUpdates和endUpdates插入对cell数量有改变的section的刷新
```
tableView.beginUpdates()
tableView.reloadSections(IndexSet([0]), with: .automatic)
tableView.endUpdates()
```
2. 在 UITableView reloadData 完后操作
```
DispatchQueue.main.async {
   tableView.beginUpdates()
   tableView.endUpdates()
}
```
