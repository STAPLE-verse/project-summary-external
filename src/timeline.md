---
theme: dashboard
title: Event Timeline
toc: false
---

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

```js
// Extract tasks and create timeline events
const tasks = jsonData.tasks ?? [];
const projectMembers = jsonData.projectMembers ?? [];
const projectName = jsonData.name || "Unnamed Project"; // Fallback if name is missing
const projectCreatedAt = jsonData.createdAt || null; // Check for project creation date

// Map projectMembers by their `id` for quick lookup
const projectMembersById = Object.fromEntries(
  projectMembers.flatMap(member =>
    (member.users ?? []).map(user => [
      member.id,
      { username: user.username, firstName: user.firstName, lastName: user.lastName }
    ])
  )
);

// Create timeline events from tasks, logs, project members, project creation, and element creation
let timelineEvents = [
  // Add project creation event
  ...(projectCreatedAt
    ? [
        {
          name: `Project Created: ${projectName}`,
          date: projectCreatedAt,
          type: "Project Creation"
        }
      ]
    : []),

  // Add task creation, deadlines, and logs
  ...tasks.flatMap(task => {
    const taskCreation = task.createdAt ? [{ name: task.name, date: task.createdAt, type: "Task Created" }] : [];
    const taskDeadline = task.deadline ? [{ name: task.name, date: task.deadline, type: "Deadline" }] : [];
    const taskLogs = (task.taskLogs ?? [])
      .filter(log => log.status !== "NOT_COMPLETED") // Exclude NOT_COMPLETED logs
      .map(log => {
        const completer = projectMembersById[log.completedById] || {};
        const completerName = [completer.firstName, completer.lastName].filter(Boolean).join(" ") || "Unknown Completer";
        return {
          name: `${task.name} (Completed by: ${completer.username || completerName})`,
          date: log.createdAt,
          type: "Task Completed"
        };
      });
    return [...taskCreation, ...taskDeadline, ...taskLogs];
  }),

  // Add element creation events
  ...tasks
    .map(task => task.element) // Extract element from each task
    .filter(element => element?.createdAt) // Ensure element and createdAt exist
    .map(element => ({
      name: element.name || "Unnamed Element", // Use element name or fallback
      date: element.createdAt,
      type: "Element Created"
    })),

  // Add project member events
  ...projectMembers.flatMap(member => {
    const users = member.users ?? []; // Access nested users array safely
    const memberName = member.name || "the project"; // Fallback to "the project" if name is missing
    const isTeam = users.length > 1; // Check if there are multiple users

    return users.map(user => {
      const username = user.username || "Unknown Username";
      const firstName = user.firstName || "";
      const lastName = user.lastName || "";
      const fullName = [firstName, lastName].filter(Boolean).join(" "); // Join only non-empty parts

      // Create the event name
      const eventName = isTeam
        ? `Team member added to ${memberName}: ${username} (${fullName || "No Name Provided"})`
        : `${username}: ${fullName || "No Name Provided"}`;

      return {
        name: eventName,
        date: member.createdAt,
        type: "Project Member Added"
      };
    });
  })
];

// Remove duplicate events based on `name`, `date`, and `type`
timelineEvents = timelineEvents.filter(
  (event, index, self) =>
    index === self.findIndex(e => e.name === event.name && e.date === event.date && e.type === event.type)
);

// Sort timeline events by date
timelineEvents.sort((a, b) => new Date(a.date) - new Date(b.date));

//format date for heatmap
timelineEvents = timelineEvents.map(event => ({
  ...event,
  dateFormat: new Date(event.date).toISOString().split("T")[0]
}));

// Create a container element for the timeline
const timelineContainer = document.createElement("div");
const list = document.createElement("ol");

// Loop through events and append them as list items
timelineEvents.forEach(event => {
  const listItem = document.createElement("li");
  listItem.innerHTML = `<strong>${event.type}:</strong> ${event.name} - ${new Date(event.date).toLocaleString()}`;
  list.appendChild(listItem);
});

// Append the list to the container
timelineContainer.appendChild(list);
```

```js
// get the whole year
function generateFullYearRange(year) {
  const start = new Date(`${year}-01-01T00:00:00Z`);
  const end = new Date(`${year}-12-31T23:59:59Z`);
  const dates = [];
  let currentDate = new Date(start);
  while (currentDate <= end) {
    dates.push(new Date(currentDate).toISOString().split("T")[0]); // Format as yyyy-mm-dd
    currentDate.setUTCDate(currentDate.getUTCDate() + 1); // Use UTC to avoid daylight saving issues
  }
  return dates;
}

// Extract earliest and latest years from timelineEvents
const startYear = new Date(Math.min(...timelineEvents.map(event => new Date(event.dateFormat)))).getFullYear();
const endYear = new Date(Math.max(...timelineEvents.map(event => new Date(event.dateFormat)))).getFullYear();

// Generate a full range for each year in the range
const fullYearRange = [];
for (let year = startYear; year <= endYear; year++) {
  fullYearRange.push(...generateFullYearRange(year));
}

// Extract years in the range
const years = Array.from(new Set(timelineEvents.map(event => new Date(event.date).getUTCFullYear())));

// Merge the full date range with heatmapData
const heatmapData = fullYearRange.map(day => {
  const eventsForDay = timelineEvents.filter(event => event.dateFormat === day); // Exact match
  return {
    day,
    count: eventsForDay.length,
    name: eventsForDay.map(event => event.name) // Collect all event names as an array
  };
});

// month labels
const monthNames = [
  "Jan", "Feb", "Mar", "Apr", "May", "Jun",
  "Jul", "Aug", "Sep", "Oct", "Nov", "Dec"
];

const monthLabels = monthNames.map((name, index) => {
  const startOfMonth = new Date(Date.UTC(startYear, index, 1));
  const middleOfMonth = new Date(startOfMonth);
  middleOfMonth.setUTCDate(Math.ceil(new Date(startYear, index + 1, 0).getUTCDate() / 2)); // Middle of the month
  return {
    date: middleOfMonth.toISOString().split("T")[0], // Middle day of the month
    label: name
  };
});

const dayLabels = fullYearRange.map(day => ({
  day: day, // The full date (YYYY-MM-DD)
  label: new Date(day).getUTCDate() // Day of the month (1, 2, ..., 31)
}));

const containerWidth = document.getElementById('plot-container').clientWidth;
// Generate year labels for each year in the heatmap range
const yearLabels = years.map(year => ({
  year: String(year), // Force it to be plain text
  x: d3.utcWeek.count(d3.utcYear(new Date(`${year}-01-01T00:00:00Z`)), new Date(`${year}-01-01T00:00:00Z`)),
}));

// Update the heatmap
const heatmapPlot = Plot.plot({
  width: containerWidth,
  // height: 400, // Adjust for the extra row for year labels
  x: { axis: null },
  y: { 
    tickFormat: (d, i) => (i === 0 ? years[0] : Plot.formatWeekday("en", "narrow")(d)), // Replace top row with year label
    tickSize: 0
  },
  fy: {
    domain: years, // Use the years as facets
    tickFormat: (year) => year, // Label each facet with the year
    label: "Year", // Add a label for the facet axis
    padding: 1 // Add space between the year blocks
  },
  color: { 
    range: ["white", ...d3.schemeSpectral[9]],
    domain: [0, Math.max(...heatmapData.map(d => d.count))]
  },
  marks: [
    // Heatmap cells
    Plot.cell(heatmapData, {
      x: (d) => d3.utcWeek.count(d3.utcYear(new Date(fullYearRange[0])), new Date(d.day)),
      y: (d) => new Date(d.day).getUTCDay() + 1, // Offset rows by 1 to make space for year
      fill: (d) => d.count,
      title: (d) => `${d.count} event(s) on ${d.day}`,
      r: 2,
    }),

    // Year label
    Plot.text(yearLabels, {
      x: (d) => d.x,
      y: 0, // Position at the top row
      text: (d) => d.year,
      fontSize: 12,
      textAnchor: "middle",
      fill: "currentColor",
    }),

    // Month labels
    Plot.text(monthLabels, {
      x: (d) => d3.utcWeek.count(d3.utcYear(new Date(fullYearRange[0])), new Date(d.date)),
      y: -1, // Adjust to match the new spacing
      text: (d) => d.label,
      fontSize: 12,
      textAnchor: "middle",
      fill: "currentColor"
    }),

    // Day numbers
    Plot.text(dayLabels, {
      x: (d) =>
        d3.utcWeek.count(d3.utcYear(new Date(fullYearRange[0])), new Date(d.day)),
      y: (d) => new Date(d.day).getUTCDay() + 1, // Adjust for the offset
      text: (d) => d.label, // Day of the month
      fontSize: 8,
      fill: "gray",
      textAnchor: "middle"
    })
  ]
});
```

## Calendar

<div id="plot-container" style="width: 100%; height: auto;">
  ${heatmapPlot}
</div>

## Timeline Data

${timelineContainer}

<style>
.hero, .grid {
  max-width: none; /* Remove the restriction on these containers */
  width: 100%; /* Allow full width */
}

.grid .card {
  max-width: none; /* Ensure cards are also unrestricted */
  margin: 1rem; /* Add spacing between cards */
}

p, table, figure, figcaption, h1, h2, h3, h4, h5, h6, ol, .katex-display {
  max-width: 100%; /* Ensure all these elements can use full width */
}
</style>