---
theme: dashboard
title: Contributors
toc: false
---

<link rel="stylesheet" href="style.css">

```js 
const jsonData = FileAttachment("./data/project_summary.json").json();
import * as Plot from "npm:@observablehq/plot";
import * as d3 from "npm:d3";

function printJsonStructure(obj, indent = 0) {
  const padding = " ".repeat(indent);
  return Object.entries(obj)
    .map(([key, value]) => {
      if (typeof value === "object" && value !== null) {
        return `${padding}${key}: {\n${printJsonStructure(value, indent + 2)}\n${padding}}`;
      } else {
        return `${padding}${key}: ${typeof value}`;
      }
    })
    .join("\n");
}
```


