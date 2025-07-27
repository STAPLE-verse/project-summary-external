---
toc: false
---

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
  localStorage.removeItem("jsonData")
  window.location.href = '/';
}}));
```

</div>

```js wait-for-data
const jsonNew = await jsonfile.json();
localStorage.setItem("jsonData", JSON.stringify(jsonNew));
window.location.href = '/';
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
  <a href="https://app.staple.science">
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

  This project summary was created using the STAPLE app. STAPLE empowers researchers to manage their projects with clarity, ensuring open and transparent documentation throughout the project lifecycle. By providing tools for seamless data and metadata tracking, STAPLE supports the principles of open science and fosters collaboration across disciplines. Learn more at <a href="https://staple.science">https://staple.science</a>.
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
```

```js download data
// Create a Blob from the JSON data
const jsonBlob = new Blob([JSON.stringify(jsonData, null, 2)], { type: "application/json" });

// Generate a temporary URL for the Blob
const jsonUrl = URL.createObjectURL(jsonBlob);

// Create a dynamic download link
const downloadLink = document.createElement("a");
downloadLink.href = jsonUrl; // Use the generated Blob URL
downloadLink.textContent = "Download Data";
downloadLink.download = "project_summary.json"; // Suggest a filename for download

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
        <p><b>Project Description:</b> ${jsonData.description || "No Description"}</p>
        <p><b>Project Metadata:</b></p>
        ${Object.entries(jsonData.metadata || {}).map(([key, value]) => html`
                  <p><strong>${toProperCase(key)}:</strong> ${value}</p>
                `)}
      </div>
    </div>
    <div class="grid grid-cols-3">
      <div class="card">
        <center><a href="people_roles">Check out the contributors</a></center>
      </div>
      <div class="card">
        <center><a href="tasks">Check out the tasks</a></center>
      </div>
      <div class="card">
        <center><a href="forms">Check out the metadata</a></center>
      </div>
      <div class="card">
        <center><a href="timeline">Check out the timeline</a></center>
      </div>
    </div>
  ` : ""
}
