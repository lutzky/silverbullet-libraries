---
tags: meta
---
# Icon for people

```space-lua
tag.define {
  name = "person",
  transform = function(o)
    o.pageDecoration = { prefix = "🧑 " }
    return o
  end
}
```

# Make sure that people have people tag

```space-lua
function personMetaData()
      return [[---
tags: person
---

]]
end

event.listen {
  name = "editor:pageCreating",
  run = function(e)
    if not e.data.name:startsWith("People/") then
      return
    end
    return {
      text = personMetaData(),
      perm = "rw"
    }
  end
}
