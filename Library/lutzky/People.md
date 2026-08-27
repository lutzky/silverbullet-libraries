---
name: Library/lutzky/People
tags: meta/library
---

# Hint about retroactively doing this

You can add tags to a pile of existing files using the `yq` utility:

```
for f in *.md
    echo $f
    if test (head -n 1 "$f") = ---
        # File already has frontmatter: Safely update/append tags using yq
        yq --front-matter=process '.tags = ((.tags // []) + "person" | unique)' -i "$f"
    else
        # File does not have frontmatter: Prepend fresh frontmatter to the file
        printf "---\ntags:\n  - person\n---\n\n" | cat - "$f" >temp.md && mv temp.md "$f"
        echo "Created fresh frontmatter for: $f"
    end
end
```

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
