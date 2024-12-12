---
theme: dashboard
title: Event Timeline
toc: false
---

```js 
// imports 
import * as Plot from "npm:@observablehq/plot";
import * as d3 from "npm:d3";

// calendar function
function calendar({
  date = Plot.identity,
  inset = 0.5,
  ...options
} = {}) {
  let D;
  return {
    fy: {transform: (data) => (D = Plot.valueof(data, date, Array)).map((d) => new Date(d).getUTCFullYear())},
    x: {transform: () => D.map((d) => d3.utcWeek.count(d3.utcYear(d), d))},
    y: {transform: () => D.map((d) => new Date(d).getUTCDay())},
    inset,
    ...options
  };
}

// month line function 
class MonthLine extends Plot.Mark {
  static defaults = {stroke: "currentColor", strokeWidth: 1};
  constructor(data, options = {}) {
    const {x, y} = options;
    super(data, {x: {value: x, scale: "x"}, y: {value: y, scale: "y"}}, options, MonthLine.defaults);
  }
  render(index, {x, y}, {x: X, y: Y}, dimensions) {
    const {marginTop, marginBottom, height} = dimensions;
    const dx = x.bandwidth(), dy = y.bandwidth();
    return htl.svg`<path fill=none stroke=${this.stroke} stroke-width=${this.strokeWidth} d=${
      Array.from(index, (i) => `${Y[i] > marginTop + dy * 1.5 // is the first day a Monday?
          ? `M${X[i] + dx},${marginTop}V${Y[i]}h${-dx}` 
          : `M${X[i]},${marginTop}`}V${height - marginBottom}`)
        .join("")
    }>`;
  }
}
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

// Generate a full range of dates for the year(s)
const startDate = d3.utcYear(new Date(d3.min(timelineEvents, (d) => new Date(d.date))));
const endDate = d3.utcYear.offset(d3.utcYear(new Date(d3.max(timelineEvents, (d) => new Date(d.date)))), 1) - 1;

// Generate all dates from startDate to endDate
const allDates = d3.utcDays(startDate, endDate);

// Map all dates to include counts, defaulting to zero for missing dates
const eventsByDateMap = new Map(eventsByDate); // Convert aggregated data to a Map for quick lookup
const completeEventsCount = allDates.map((date) => ({
  date: date.toISOString().split("T")[0],
  count: eventsByDateMap.get(date.toISOString().split("T")[0]) || 0
}));

const heatmapPlot = (() => {
  const start = d3.utcDay.offset(d3.min(completeEventsCount, (d) => new Date(d.date)));
  const end = d3.utcDay.offset(d3.max(completeEventsCount, (d) => new Date(d.date)));

  // Dynamically calculate width
  const containerWidth = document.getElementById("plot-container").clientWidth || 1152;

  return Plot.plot({
    width: containerWidth, // Use dynamic width
    height: d3.utcYear.count(start, end) * 200,
    axis: null,
    padding: 0,
    x: {
      domain: d3.range(53) // 53 weeks in a year
    },
    y: {
      axis: "left",
      domain: [-1, 0, 1, 2, 3, 4, 5, 6], // Include all days of the week
      ticks: [0, 1, 2, 3, 4, 5, 6], // Add ticks for all days
      tickSize: 0,
      tickFormat: Plot.formatWeekday() // Format as weekdays
    },
    fy: {
      padding: 0.1,
      reverse: true
    },
    color: {
      scheme: "blues",
      domain: [0, d3.max(completeEventsCount, (d) => d.count)], // Dynamic color scaling
      legend: true,
      label: "Number of Events"
    },
    marks: [
      // Cell marks for event counts
      Plot.cell(
        completeEventsCount,
        calendar({
          date: (d) => new Date(d.date),
          fill: (d) => d.count, // Color cells by the number of events
          title: (d) => `${d.date}: ${d.count} event${d.count !== 1 ? "s" : ""}` // Tooltip content
        })
      ),

      // Text labels for the day of the month
      Plot.text(
        completeEventsCount,
        calendar({
          date: (d) => new Date(d.date),
          text: (d) => new Date(d.date).getUTCDate(), // Day of the month
          frameAnchor: "middle", // Center the text
          fill: "black", // Text color
          title: (d) => `${d.date}: ${d.count} event${d.count !== 1 ? "s" : ""}` // Tooltip content
        })
      ),
      // Year and month labels (as before)
      Plot.text(
        d3.utcYears(d3.utcYear(start), end),
        calendar({ text: d3.utcFormat("%Y"), frameAnchor: "right", x: 0, y: -1, dx: -20 })
      ),
      Plot.text(
        d3.utcMonths(d3.utcMonth(start), end).map(d3.utcMonday.ceil),
        calendar({ text: d3.utcFormat("%b"), frameAnchor: "left", y: -1 })
      )
    ]
  });
})();
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