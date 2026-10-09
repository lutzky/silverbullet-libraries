---
name: Library/lutzky/Remaining tasks
tags: meta/library
---

To disable, set front-matter `pageDecoration.disableRemainingTasks` to `true`.

# Example

* [ ] task1
* [x] task2
* [ ] task3

# Implementation

```space-lua
local function remainingTasksWidget()
  local tasks = query[[from t = index.tasks()
    where not t.done and
    t.page == editor.getCurrentPage()
    order by t.pos asc
    select templates.taskItem(t)
  ]]
  if #tasks > 0 then
    return table.concat(tasks)
  end
end
  
view.define {
  name = "lutzky.remainingTasks",
  title = "✅ Remaining Tasks",
  command = "Navigate: Remaining Tasks",
  dock = "page-top",
  frame = "minimal",
  content = remainingTasksWidget,
  defaultOpen = true,
  refreshOn = { 
    -- 2.11
    "editor:pageLoaded",
    "mq:emptyQueue:indexQueue",
    -- 2.12
    "navigate",
    "index",
  },
  refreshOnOpen = true,
  supportedDocks = { "page-top", "bhs", "rhs", "lhs", "page-bottom" },
}
```
