---
toc: false
---

<link rel="stylesheet" href="style.css">

```js
const jsonData = FileAttachment("./data/project_summary.json").json();
```

```js
// Function to convert a string to Proper Case
const toProperCase = (str) =>
  str.replace(/\b\w+/g, (word) => word.charAt(0).toUpperCase() + word.slice(1));

// Create a container for project metadata
const metadataContainer = document.createElement("div");
metadataContainer.className = "metadata-container"; // Optional: Add a class for styling

// Loop through the metadata and add key-value pairs as HTML elements
Object.entries(jsonData.metadata || {}).forEach(([key, value]) => {
  const metadataItem = document.createElement("p");
  metadataItem.innerHTML = `<strong>${toProperCase(key)}:</strong> ${value}`;
  metadataContainer.appendChild(metadataItem);
});

// Append the metadata container to the desired location in the DOM
document.body.appendChild(metadataContainer); // Replace `document.body` with your target container

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

<div class="hero">
  <h1>${jsonData.name || "Unnamed Project"}</h1>
</div>

<div class="flex flex-row">
  <div class="card">
    <b>Project Start Date:</b> ${formatDate(jsonData.createdAt)}
    <p><b>Project Metadata and Settings Last Update:</b> ${formatDate(jsonData.updatedAt)}</p>
    <p><b>Project Description:</b> ${jsonData.description || "No Description"}</p>
    <p><b>Project Metadata:</b></p>
    ${metadataContainer}
  </div>
</div>

<div class="grid grid-cols-4">
  <div class="card">
    <center><a href="people_roles">Check out the contributors</a></center>
  </div>
  <div class="card">
    <center><a href="elements_tasks">Check out the elements and tasks</a></center>
  </div>
  <div class="card">
    <center><a href="forms">Check out the metadata</a></center>
  </div>
  <div class="card">
    <center><a href="timeline">Check out the timeline</a></center>
  </div>
    <div class="card">
    <center><a href="./_file/data/project_summary.json">Download Data</a></center>
  </div>
</div>
