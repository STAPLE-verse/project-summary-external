---
theme: dashboard
title: Contributors
toc: false
---

```js 
const jsonData = FileAttachment("./data/project_summary.json").json();
import * as Plot from "npm:@observablehq/plot";
import * as d3 from "npm:d3";
import * as Inputs from "npm:@observablehq/inputs";
```

```js
// Extract project members where there is exactly one user in the `users` array
const filteredMembers = jsonData.projectMembers.filter(
  (member) => member.name === null
);

// Map the data into a dataframe-like array of objects
const projectMembersDataFrame = filteredMembers.map((member) => {
  const user = member.users[0];
  const roles = member.roles.map((role) => role.name).join(", ");
  return {
    projectMemberId: member.id,
    username: user.username || "Not Provided",
    firstName: user.firstName || "Not Provided",
    lastName: user.lastName || "Not Provided",
    roles: roles || "No Roles",
    createdAt: member.createdAt,
    updatedAt: member.updatedAt,
  };
});

// Extract project members where there is more than one user in the `users` array
const groupMembers = jsonData.projectMembers.filter(
  (member) => member.name !== null
);

// Map the data into a dataframe-like array of objects, each user gets their own row
const groupMembersDataFrame = groupMembers.flatMap((member) => {
  return member.users.map((user) => ({
    projectMemberId: member.id,
    groupName: member.name || "Unnamed Group",
    username: user.username || "Not Provided",
    firstName: user.firstName || "Not Provided",
    lastName: user.lastName || "Not Provided",
    roles: member.roles.map((role) => role.name).join(", ") || "No Roles",
    createdAt: member.createdAt,
    updatedAt: member.updatedAt,
  }));
});

// Extract all roles from project members
const allRoles = jsonData.projectMembers.flatMap((member) => member.roles);

// Deduplicate roles based on `name`, `description`, and `taxonomy`
const uniqueRolesDataFrame = Array.from(
  new Map(
    allRoles.map((role) => [
      `${role.name}-${role.description}-${role.taxonomy}`, // Unique key for deduplication
      role
    ])
  ).values()
);

// Transform the unique roles into a dataframe-like structure
const rolesDataFrame = uniqueRolesDataFrame.map((role) => ({
  name: role.name || "No Name",
  description: role.description || "No Description",
  taxonomy: role.taxonomy || "No Taxonomy",
}));

// Extract tasks data and expand to include task logs
const tasksDataFrame = jsonData.tasks.flatMap((task) =>
  task.taskLogs
    .filter((log) => log.completedById !== null) // Skip logs where completedById is null
    .map((log) => ({
      taskId: task.id,
      createdAt: task.createdAt,
      updatedAt: task.updatedAt,
      createdById: task.createdById,
      formVersionId: task.formVersionId || "Not Provided",
      deadline: task.deadline || "No Deadline",
      name: task.name || "Unnamed Task",
      description: task.description || "No Description",
      status: task.status || "Unknown Status",
      elementName: task.element?.name || "No Element Name",
      elementDescription: task.element?.description || "No Element Description",
      taskLogCreatedAt: log.createdAt,
      taskLogStatus: log.status || "Unknown Status",
      taskLogMetadata: log.metadata || "No Metadata",
      completedById: log.completedById,
      assignedToId: log.assignedToId,
    }))
);
```

```js
// Deduplicate task logs to only include the latest log for each combination of `taskId` and `assignedToId`
const latestTaskLogs = Array.from(
  tasksDataFrame
    .reduce((map, log) => {
      const key = `${log.taskId}-${log.assignedToId}`;
      const existingLog = map.get(key);

      // Keep the log with the latest `createdAt`
      if (!existingLog || new Date(log.taskLogCreatedAt) > new Date(existingLog.taskLogCreatedAt)) {
        map.set(key, log);
      }

      return map;
    }, new Map())
    .values()
);
// Group task logs by user and count completed tasks and metadata forms
const individualsWithTaskData = projectMembersDataFrame.map((individual) => {
  // Filter task logs completed by the individual
  const completedTaskLogs = latestTaskLogs.filter(
    (log) => log.assignedToId === individual.projectMemberId && log.taskLogStatus === "COMPLETED"
  );

  // Count completed tasks
  const numberOfTasksCompleted = completedTaskLogs.length;

  // Count metadata forms
  const numberOfMetadataForms = completedTaskLogs.filter(
    (log) => log.taskLogMetadata !== "No Metadata"
  ).length;

  // Construct name, replacing "Not Provided Not Provided" with "No Name Provided"
  const fullName =
    `${individual.firstName} ${individual.lastName}`.trim() === "Not Provided Not Provided"
      ? "No Name Provided"
      : `${individual.firstName} ${individual.lastName}`;

  return {
    projectMemberId: individual.projectMemberId,
    name: `${fullName} (${individual.username})`,
    numberOfTasksCompleted,
    numberOfMetadataForms,
  };
});

const style = document.createElement("style");
style.textContent = `
  .individual-cards-container {
    display: flex;
    flex-wrap: wrap; /* Allows cards to wrap if the row gets too long */
    gap: 16px; /* Adds spacing between cards */
    justify-content: flex-start; /* Align cards to the start of the row */
  }
  .individual-card {
    border: 1px solid #ccc;
    border-radius: 8px;
    padding: 16px;
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
    width: 300px;
    font-family: Arial, sans-serif;
    flex: 1 1 calc(33% - 16px); /* Adjust to fit 3 cards per row, with gaps */
    box-sizing: border-box; /* Ensure padding doesn't exceed width */
  }
  .individual-card h4 {
    margin-top: 0;
    font-size: 1.2em;
  }
  .individual-card p {
    margin: 4px 0;
  }
`;
document.head.appendChild(style);

// Function to render individual cards in a flex container
function renderIndividualCards(data) {
  return html`<div class="individual-cards-container">
    ${data.map(
      (individual) => html`
        <div class="individual-card">
          <h4>${individual.name}</h4>
          <p><strong>Tasks Completed:</strong> ${individual.numberOfTasksCompleted}</p>
          <p><strong>Metadata Forms:</strong> ${individual.numberOfMetadataForms}</p>
        </div>
      `
    )}
  </div>`;
}
```

```js
// Group task logs by team ID (assignedToId) and calculate completed tasks and metadata forms
const teamsWithTaskData = groupMembers.reduce((teamsMap, team) => {
  // Initialize or update team entry
  const teamData = teamsMap.get(team.id) || {
    teamId: team.id,
    teamName: team.name || "Unnamed Team",
    memberNames: [],
    numberOfTasksCompleted: 0,
    numberOfMetadataForms: 0,
  };

  // Add team member names
  team.users.forEach((user) => {
    const memberName = `${user.firstName?.trim() || "No Name"} ${user.lastName?.trim() || "Provided"} (${user.username || "No Username"})`;
    if (!teamData.memberNames.includes(memberName)) {
      teamData.memberNames.push(memberName);
    }
  });

  // Filter task logs completed by this team
  const completedTaskLogs = latestTaskLogs.filter(
    (log) => log.assignedToId === team.id && log.taskLogStatus === "COMPLETED"
  );

  // Update team-level task counts
  teamData.numberOfTasksCompleted += completedTaskLogs.length;
  teamData.numberOfMetadataForms += completedTaskLogs.filter(
    (log) => log.taskLogMetadata !== "No Metadata"
  ).length;

  // Store updated team data
  teamsMap.set(team.id, teamData);

  return teamsMap;
}, new Map());

// Convert the teams map to an array for rendering
const teamsDataArray = Array.from(teamsWithTaskData.values());

// Function to render team cards in a flex container
function renderTeamCards(data) {
  return html`<div class="team-cards-container">
    ${data.map(
      (team) => html`
        <div class="team-card">
          <h4>${team.teamName}</h4>
          <p><strong>Members:</strong> ${team.memberNames.join(", ")}</p>
          <p><strong>Tasks Completed:</strong> ${team.numberOfTasksCompleted}</p>
          <p><strong>Metadata Forms:</strong> ${team.numberOfMetadataForms}</p>
        </div>
      `
    )}
  </div>`;
}

// Apply flexbox styling for team cards (similar to individual cards)
style.textContent += `
  .team-cards-container {
    display: flex;
    flex-wrap: wrap;
    gap: 16px;
    justify-content: flex-start;
  }
  .team-card {
    border: 1px solid #ccc;
    border-radius: 8px;
    padding: 16px;
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
    width: 300px;
    font-family: Arial, sans-serif;
    flex: 1 1 calc(50% - 16px); /* Adjust to fit 2 cards per row, with gaps */
    box-sizing: border-box;
  }
  .team-card h4 {
    margin-top: 0;
    font-size: 1.4em;
  }
  .team-card p {
    margin: 4px 0;
  }
`;
```

## Individual Members
<p>

${renderIndividualCards(individualsWithTaskData)}

## Teams
<p>

${renderTeamCards(teamsDataArray)}

## View Data by Member or Team
<p>

```js
const tasksWithNames = tasksDataFrame.map((task) => {
  // Find the assigned team or individual name
  const assignedName = (() => {
    const assignedTeam = groupMembersDataFrame.find(
      (team) => team.projectMemberId === task.assignedToId
    );

    if (assignedTeam) {
      return assignedTeam.groupName || "Unnamed Team";
    }

    const assignedMember = projectMembersDataFrame.find(
      (member) => member.projectMemberId === task.assignedToId
    );

    if (assignedMember) {
      const fullName =
        `${assignedMember.firstName?.trim() || "Not Provided"} ${
          assignedMember.lastName?.trim() || "Not Provided"
        }`.trim();
      return fullName === "Not Provided Not Provided"
        ? `No Name Provided (${assignedMember.username || "No Username"})`
        : `${fullName} (${assignedMember.username || "No Username"})`;
    }
    return "Unassigned"; // Fallback if no match is found
  })();

  // Find the completed by team or individual name
  const completedByName = (() => {
    const completedByTeam = groupMembersDataFrame.find(
      (team) => team.projectMemberId === task.completedById
    );

    if (completedByTeam) {
      return completedByTeam.groupName || "Unnamed Team";
    }

    const completedByMember = projectMembersDataFrame.find(
      (member) => member.projectMemberId === task.completedById
    );

    if (completedByMember) {
      const fullName =
        `${completedByMember.firstName?.trim() || "Not Provided"} ${
          completedByMember.lastName?.trim() || "Not Provided"
        }`.trim();
      return fullName === "Not Provided Not Provided"
        ? `No Name Provided (${completedByMember.username || "No Username"})`
        : `${fullName} (${completedByMember.username || "No Username"})`;
    }
    return "Unknown"; // Fallback if no match is found
  })();

  // Return a new object with updated assignedTo and completedBy fields
  return {
    ...task, // Spread the existing task data
    assignedTo: assignedName, // Replace assignedToId with the formatted name
    completedBy: completedByName, // Replace completedById with the formatted name
  };
});

const selectDropDown = view(
  Inputs.select(
Array.from(
      new Set(tasksWithNames.map((task) => task.assignedTo)) 
    ),
  {     
    sort: true,
    unique: true,
    value: "Select a Member or Team", // Default selection (empty)
  }
))
```

```js
const tableOutput = view(
    Inputs.table(
        tasksWithNames.filter((d) => d.assignedTo === selectDropDown)),
        { sort: "name", select: false }
        )
```

```js
function createHtmlTable(data, containerId, tableId) {
  const container = document.getElementById(containerId);

  if (!container) {
    console.error(`Container with id "${containerId}" not found.`);
    return;
  }

  // Create the table
  const table = document.createElement("table");
  table.id = tableId; // Assign the provided tableId
  table.className = "custom-table";

  // Create table header
  const thead = document.createElement("thead");
  const headerRow = document.createElement("tr");

  // Use the keys of the first object in the data as headers
  const headers = Object.keys(data[0]);
  headers.forEach((key) => {
    const th = document.createElement("th");
    th.textContent = key.replace(/([A-Z])/g, " $1"); // Add spaces before uppercase letters
    headerRow.appendChild(th);
  });

  thead.appendChild(headerRow);
  table.appendChild(thead);

  // Create table body
  const tbody = document.createElement("tbody");

  data.forEach((item) => {
    const row = document.createElement("tr");
    headers.forEach((key) => {
      const td = document.createElement("td");
      td.textContent = item[key] || "N/A"; // Fallback to "N/A" if the value is null/undefined
      row.appendChild(td);
    });
    tbody.appendChild(row);
  });

  table.appendChild(tbody);

  // Clear any existing content in the container and add the table
  container.innerHTML = "";
  container.appendChild(table);
}

// Call the function to create the table
const filteredData = tasksWithNames.filter(
  (task) => task.assignedTo === selectDropDown
);

createHtmlTable(filteredData, "table-container", "tasks-table");

$("#tasks-table").DataTable({
  paging: true,
  searching: true,
  ordering: true,
  responsive: true,
  scrollX: true,
  initComplete: function () {
    // Add a text input for each column
    this.api()
      .columns()
      .every(function () {
        const column = this;
        const header = $(column.header());
        const input = $('<input type="text" placeholder="Search ' + header.text() + '" />')
          .appendTo($(header).empty())
          .on("keyup change clear", function () {
            if (column.search() !== this.value) {
              column.search(this.value).draw();
            }
          });
      });
  },
});
```

<p>

## Data Table Version

<!-- Load jQuery -->
<script src="https://code.jquery.com/jquery-3.6.0.min.js"></script>

<!-- Load DataTables -->
<link rel="stylesheet" href="https://cdn.datatables.net/1.13.6/css/jquery.dataTables.min.css">
<script src="https://cdn.datatables.net/1.13.6/js/jquery.dataTables.min.js"></script>


<div id="table-container"></div>


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