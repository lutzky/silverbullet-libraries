---
name: Library/lutzky/Projects
tags: meta/library
---

Transclude the "Projects" section below like so:

```
![[^Library/lutzky/Projects#Dashboard]]
```

This gives you a "mostly read-only" homepage, while keeping this section easy to edit.

# Dashboard

## Inbox notes (${#inbox.notes()})
${some(query[[from p = inbox.notes() select inbox.template(p)]]) or "*None*"}

## Tasks
These are all open tasks tagged `#next`.
${some(query[[
    from p = index.tasks("next") where p.done == false
    select templates.taskItem(p)
  ]]) or "*None*"}

${projects.render_dashboard()}

# Code for main project list
```space-lua
projects = projects or {}

function projects.load_dashboard_data()
  local all_projects = query[[from index.pages() 
    where string.startsWith(name, "Projects/") 
    and status != "Done" 
    and status != "Decided no"
  ]]
  
  local buckets = {
    active = {},
    snoozed = {},
    ready = {},
    blocked = {},
    snoozed_reminders = {}
  }
  
  local today = os.date("%Y-%m-%d")
  
  for _, p in ipairs(all_projects) do
    local is_snoozed = type(p.snooze_until) == "string" and p.snooze_until > today
    local is_reminder = table.includes(p.tags or {}, "reminder")
    
    if p.status == "Active" then
      if is_snoozed then
        if is_reminder then
          table.insert(buckets.snoozed_reminders, p)
        else
          table.insert(buckets.snoozed, p)
        end
      else
        table.insert(buckets.active, p)
      end
    elseif p.status == "Ready" then
      table.insert(buckets.ready, p)
    elseif p.status == "Blocked" then
      table.insert(buckets.blocked, p)
    end
  end
  
  local function sort_by_priority(a, b) 
    return (a.priority or "P9") < (b.priority or "P9") 
  end
  local function sort_by_snooze(a, b) 
    return (a.snooze_until or "") < (b.snooze_until or "") 
  end
  
  table.sort(buckets.active, sort_by_priority)
  table.sort(buckets.ready, sort_by_priority)
  table.sort(buckets.blocked, sort_by_priority)
  table.sort(buckets.snoozed, sort_by_snooze)
  table.sort(buckets.snoozed_reminders, sort_by_snooze)
  
  return buckets
end

function projects.render_bucket(bucket_list, max_items)
  if #bucket_list == 0 then return "*None*\n" end
  
  local out = ""
  local count = max_items and math.min(#bucket_list, max_items) or #bucket_list

  for i = 1, count do
    local p = bucket_list[i]
    local project_entry = (
      projects.list_snooze_prefix(p.snooze_until) ..
      projects.list_priority_string(p.priority) .. " " ..
      "[[" .. p.name .. "]] " ..
      lutzky_utils.list_tagify(p.tags) ..
      "\n"
    )
    out = out .. project_entry
  end
  return out
end

function projects.render_dashboard()
  local data = projects.load_dashboard_data()
  
  local out = ""
  
  out = out .. string.format("## Active (%d)\n", #data.active)
  out = out .. projects.render_bucket(data.active) .. "\n"

  out = out .. string.format("## Snoozed (%d)\n", #data.snoozed)
  out = out .. projects.render_bucket(data.snoozed, 10)

  if #data.snoozed > 10 then
    out = out .. "... (see more below)"
  end

  out = out .. "\n"

  out = out .. string.format("## Ready (%d)\n", #data.ready)
  out = out .. projects.render_bucket(data.ready, 20)
  if #data.ready > 20 then
    out = out .. "\nMore: [[Ready Projects]]\n"
  end
  out = out .. "\n"
  
  out = out .. string.format("## Blocked (%d)\n", #data.blocked)
  out = out .. projects.render_bucket(data.blocked) .. "\n"

  out = out .. string.format("## All snoozed (%d)\n", #data.snoozed)
  out = out .. projects.render_bucket(data.snoozed) .. "\n"
  
  out = out .. string.format("## Snoozed reminders (%d)\n", #data.snoozed_reminders)
  out = out .. projects.render_bucket(data.snoozed_reminders, 10) .. "\n"

  return out
end

function projects.is_snoozed(snooze_until)
  if type(snooze_until) != "string" then
    return false
  end
  if not lutzky_utils.is_date(snooze_until) then
    -- We'll notice that these didn't get snoozed, as they'll
    -- be in the active list.
    return false
  end
  return snooze_until > os.date("%Y-%m-%d")
end

function projects.is_reminder(project)
  return table.includes(project.tags, "reminder")
end

function projects.list_priority_string(priority)
  if priority == nil then
    return "🤷"
  end
  local priority_icon = ({
    ["P0"]="🟥",
    ["P1"]="🟨",
    ["P2"]="🟩",
  })[priority] or "🤷"
  return priority_icon .. priority
end

function projects.list_snooze_prefix(snooze_until)
  if not projects.is_snoozed(snooze_until) then
    return ""
  end
  return "😴" .. snooze_until .. " "
end

projects.project_template = template.new[==[${projects.list_snooze_prefix(snooze_until)}${projects.list_priority_string(priority)} [[${name}]] ${lutzky_utils.list_tagify(tags)}]==]
```

# Validator hook
```space-lua
local badProjectStatesTemplate = template.new[==[* Page [[${name}]] has status `${status}`
]==]
local function badProjectStatesWidget()
  local badProjectStates = query[[from index.tag "page" where
    string.startsWith(name, "Projects/") and
    not table.includes({
      "Active",
      "Ready",
      "Blocked",
      "Decided no",
      "Done"
    }, status)
  ]]
  if #badProjectStates > 0 then
    return widget.new {
      markdown="# ⚠️ Bad project states\n" .. template.each(badProjectStates, badProjectStatesTemplate)
    }
  end
end
  
event.listen {
  name = "hooks:renderTopWidgets",
  run = badProjectStatesWidget
}
```

# New project creation
```space-lua
command.define {
  name = "New Project",
  run = function()
    local projectName = editor.prompt("Project name")
    if projectName and projectName != "" then
      editor.navigate("Projects/" .. projectName)
    end
  end
}

function projectMetaData()
      return [[---
priority: P2
status: Active
# pageDecoration.prefix: ""
---

# ]] .. os.date("%Y-%m-%d") .. [[
]]
end

event.listen {
  name = "editor:pageCreating",
  run = function(e)
    if not e.data.name:startsWith("Projects/") then
      return
    end
    return {
      text = projectMetaData(),
      perm = "rw"
    }
  end
}

slashCommand.define {
  name = "project",
  run = function()
    editor.insertAtPos(projectMetaData() .. "\n\n", 0)
  end
}
```

# Snoozing
```space-lua
command.define {
  name = "Project: Snooze",
  run = function()
    local page_name = editor.getCurrentPage()
  
    if not string.match(page_name, "^Projects/") then
      editor.flashNotification("Not snoozing a non-projects page")
      return
    end

    target = dateTools.selectDate()
    if target == nil then
      return
    end
    
    local current_text = editor.getText()
    local updated_text = index.patchFrontmatter(current_text, {
      {
        op = "set-key",
        path = "snooze_until",
        value = target.value,
      }
    })
    editor.setText(updated_text)
  end,
}
```

# Inbox notes

```space-lua
inbox = inbox or {}

inbox.template = template.new '**[[${name}|${string.sub(name,7)}]]** - ${string.split(space.readPage(name),"\n")[1]}'
inbox.template = template.new '**[[${name}|${string.sub(name,7)}]]** - ${inbox.firstLine(name)}'

function inbox.firstLine(pageName)
  return string.split(space.readPage(pageName), "\n")[1]
end

function inbox.notes()
  return query[[
    from index.tag "page"
    where string.startsWith(name, "Inbox/")
  ]]
end

local inboxBottomTemplate = template.new[==[This is an inbox page
]==]

local function inboxBottomWidget()
  if not string.startsWith(editor.getCurrentPage(), "Inbox/") then
    return nil
  end
  return widgets.commandButton("Delete this note", "Page: Delete")
end
  
event.listen {
  name = "hooks:renderBottomWidgets",
  run = inboxBottomWidget
}
```
