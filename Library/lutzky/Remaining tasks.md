---
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
  local tasks = query[[from index.tag "task" where
    not done and
    page == editor.getCurrentPage()
    order by pos asc
  ]]
  if #tasks > 0 then
    return widget.new {
      markdown="# ✅ Remaining Tasks\n" .. template.each(tasks,templates.taskItem)
    }
  end
end
  
-- event.listen {
--   name = "hooks:renderTopWidgets",
--   run = remainingTasksWidget
-- }

event.listen {
  name = "hooks:renderTopWidgets",
  run = function(e)
    local pageText = editor.getText()
    local fm = index.extractFrontmatter(pageText)
    if fm.frontmatter.pageDecoration and fm.frontmatter.pageDecoration.disableRemainingTasks then
      return
    end
    return remainingTasksWidget()
  end
}
```
