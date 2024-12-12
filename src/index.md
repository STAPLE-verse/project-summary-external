---
toc: false
---

```js
const jsonData = FileAttachment("./data/project_summary.json").json();
```

```js
const projectName = jsonData.name || "Unnamed Project";
```

<div class="hero">
  <h1>${projectName}</h1>
</div>

<div class="grid grid-cols-4">
  <div class="card">
    Check out the contributors: LINK
  </div>
  <div class="card">
    Check out the tasks: LINK
  </div>
  <div class="card">
    Check out the elements: LINK
  </div>
</div>

<style>

.hero {
  display: flex;
  flex-direction: column;
  align-items: center;
  font-family: var(--sans-serif);
  margin: 2rem 0 2rem;
  text-wrap: balance;
  text-align: center;
}

.hero h1 {
  font-size: 50px;
  font-weight: 900;
  background: linear-gradient(30deg, var(--theme-foreground-focus), currentColor);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.hero h2 {
  margin: 0;
  max-width: 34em;
  font-size: 20px;
  font-style: initial;
  font-weight: 500;
  line-height: 1.5;
  color: var(--theme-foreground-muted);
}

</style>
