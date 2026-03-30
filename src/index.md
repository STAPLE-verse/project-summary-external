---
toc: false
---

```js imports 
import { marked } from "npm:marked"
```

<link href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css" rel="stylesheet">
<link rel="stylesheet" href="style.css">

<div class="card" label="Use your data">

<b>To view a project summary, upload the JSON data from the project below. </b>

```js get-data
let jsonData = JSON.parse(localStorage.getItem("jsonData"))

const jsonfile = view(Inputs.file({label: "Upload the project JSON file:", accept: ".json", required: true}));
```

```js update-data
const reset = view(Inputs.button("Reset", {label: "Clear dashboard:", reduce: () => {
  localStorage.removeItem("jsonData");
  // bump a version token so other pages know things changed
  localStorage.setItem("jsonData_version", String(Date.now()));
  // force a hard navigation to the current directory's index (avoids SPA routing)
  window.location.assign(".");
}}));
```

</div>

```js wait-for-data
const jsonNew = await jsonfile.json();
localStorage.setItem("jsonData", JSON.stringify(jsonNew));
window.location.reload();
```

${
  jsonData ? html`
    <div class="hero">
      <h1>${jsonData.name || "Unnamed Project"}</h1>
    </div>
    ` : "" }

<div class="flex flex-row">
  <div class="card">

  <div class = "statistics-container">
  <a href="https://app.staplescience.com">
  <picture>
    <source
      srcSet="img/logo_white_big.png"
      media="(prefers-color-scheme: dark)"
      width=200
    />
    <img src="img/logo_black_big.png" alt="STAPLE Logo" width=200 />
  </picture>
  </a>
  </div>

  This project summary was created using the STAPLE app. STAPLE empowers researchers to manage their projects with clarity, ensuring open and transparent documentation throughout the project lifecycle. By providing tools for seamless data and metadata tracking, STAPLE supports the principles of open science and fosters collaboration across disciplines. Learn more at <a href="https://staplescience.com">https://staplescience.com</a>.
  </div>
</div>

```js functions
// Function to convert a string to Proper Case
const toProperCase = (str) =>
  str.replace(/\b\w+/g, (word) => word.charAt(0).toUpperCase() + word.slice(1));

// Function to format dates
const formatDate = (dateString) => {
  const date = new Date(dateString);
  return date.toLocaleString("en-US", {
    month: "long",
    day: "numeric",
    year: "numeric",
    hour: "numeric",
    minute: "2-digit",
    hour12: true,
  });
};

// Helper: render Markdown to HTML string via marked
function renderMarkdownToHtml(text) {
  const src = text || ""
  try {
    return marked.parse(src)
  } catch (e) {
    console.error("Markdown parse error:", e)
    return src
  }
}

// Helper: render Markdown to a DOM node (avoids html.raw)
function renderMarkdownNode(text) {
  const wrapper = document.createElement("div")
  try {
    wrapper.innerHTML = marked.parse(text || "")
  } catch (e) {
    console.error("Markdown parse error:", e)
    wrapper.textContent = text || ""
  }
  return wrapper
}
```

```js download data
// Create a Blob from the JSON data
const jsonBlob = new Blob([JSON.stringify(jsonData, null, 2)], { type: "application/json" });

// Generate a temporary URL for the Blob
const jsonUrl = URL.createObjectURL(jsonBlob);

// Create a dynamic download link
const downloadLink = document.createElement("a");
// Use the generated Blob URL
downloadLink.href = jsonUrl; 
// Text for the link
downloadLink.textContent = "Download Data";
// Suggest a filename for download
downloadLink.download = "project_summary.json"; 

// Insert the link into the appropriate DOM element
const linkContainer = document.querySelector("#dynamic-download");
if (linkContainer) {
  linkContainer.appendChild(downloadLink);
}
```

${
  jsonData ? html`
    <div class="flex flex-row">
      <div class="card">
        <p><b>Project Start Date:</b> ${formatDate(jsonData.createdAt)}</p>
        <p><b>Project Metadata and Settings Last Update:</b> ${formatDate(jsonData.updatedAt)}</p>
        <div><b>Project Description:</b></div>
        ${jsonData.description
          ? renderMarkdownNode(jsonData.description)
          : html`<p>No Description</p>`}
        <p><b>Project Metadata:</b></p>
        ${Object.entries(jsonData.metadata || {}).map(([key, value]) => html`
                  <p><strong>${toProperCase(key)}:</strong> ${value}</p>
                `)}
      </div>
    </div>
    <div class="grid grid-cols-3">
      <div class="card">
        <center><a href="Contributors">Check out the contributors</a></center>
      </div>
      <div class="card">
        <center><a href="Tasks">Check out the tasks</a></center>
      </div>
      <div class="card">
        <center><a href="Form_Data">Check out the metadata</a></center>
      </div>
      <div class="card">
        <center><a href="Events">Check out the timeline</a></center>
      </div>
    </div>
  ` : ""
}
