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
  dateFormat: new Date(event.date).toISOString().split("T")[0] // Normalize to YYYY-MM-DD
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

// Merge the full date range with heatmapData
const heatmapData = fullYearRange.map(day => {
  const eventsForDay = timelineEvents.filter(event => event.dateFormat === day); // Exact match
  return {
    day,
    count: eventsForDay.length,
    name: eventsForDay.map(event => event.name) // Collect all event names as an array
  };
});

console.log(generateFullYearRange(2024)); // Inspect all dates for 2024

const heatmapPlot = Plot.plot({
  padding: 0,
  aspectRatio: 1,
  x: { axis: null },
  y: {tickFormat: Plot.formatWeekday("en", "narrow"), tickSize: 0},
  fy: { tickFormat: "", padding: 0.1 },
  color: { scheme: "PiYG", domain: [0, 10] },
  marks: [
    Plot.cell(heatmapData, {
      x: (d) => d3.utcWeek.count(d3.utcYear(new Date(d.day)), new Date(d.day)), // Week number within year
      y: (d) => new Date(d.day).getUTCDay(), // Weekday number
      fy: (d) => "" + new Date(d.day).getUTCFullYear(), // Year
      fill: (d) => d.count, // Number of events
      title: (d) => `${d.count} event(s) on ${d.day}\n\n${d.name.join("\n")}`, // Tooltip
      inset: 0.5,
      tip: true
    })
  ]
});
```

## Calendar

<style>
  .heatmap-container {
    width: 80%; /* Reduce overall width */
    height: 400px; /* Constrain height */
    margin: auto; /* Center the heatmap */
    overflow: hidden; /* Prevent overflow issues */
  }
</style>

${heatmapPlot}

## Timeline Data

${timelineContainer}