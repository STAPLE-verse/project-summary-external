---
theme: dashboard
title: Event Timeline
toc: false
---

```js 
// imports 
import * as Plot from "npm:@observablehq/plot";
import * as d3 from "npm:d3";
import CalHeatmap from "npm:cal-heatmap";
```

```js
//data
const jsonData = FileAttachment("./data/project_summary.json").json();
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
// Aggregate events by date
const eventsByDate = d3.rollups(
  timelineEvents,
  (v) => v.length, // Count the number of events on each date
  (d) => d.dateFormat // Group by the formatted date
);

const formattedEvents = eventsByDate.map(([date, value]) => ({
  date,
  value
}));
```

## Calendar

<p>

<link rel="stylesheet" href="https://unpkg.com/cal-heatmap@4.2.4/dist/cal-heatmap.css">

```js
const cal = new CalHeatmap();
cal.paint({
  itemSelector: '#cal-heatmap',
  data: {
    source: formattedEvents,
    x: 'date', 
    y: 'value',
    }, 
  range: 12, 
  domain: { 
    type: 'month', 
    sort: 'asc', 
    label: { 
      position: 'top',
      textAlign: 'middle'
     } },
  subDomain: { 
    type: 'day',
    label: 'D' },
  scale: {
    color: {
      scheme: 'Greens',
      type: 'linear',
      domain: [0, d3.max(formattedEvents, d => d.value)],
    } },
    date: { 
      start: new Date(d3.min(formattedEvents, d => new Date(d.date))) 

      }
});
```

<div id="cal-heatmap"></div>

<p>

## Timeline Data

<div>${timelineContainer}</div>

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