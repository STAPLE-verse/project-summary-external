---
title: Event Timeline
toc: false
---

<link rel="stylesheet" href="https://unpkg.com/cal-heatmap@4.2.4/dist/cal-heatmap.css">
<script src="https://d3js.org/d3.v7.min.js"></script>
<script src="https://unpkg.com/cal-heatmap@4.2.4/dist/cal-heatmap.min.js"></script>
<script src="https://code.jquery.com/jquery-3.6.0.min.js"></script>
<link rel="stylesheet" href="https://cdn.datatables.net/1.13.6/css/jquery.dataTables.min.css">
<script src="https://cdn.datatables.net/1.13.6/js/jquery.dataTables.min.js"></script>
<link rel="stylesheet" href="https://cdn.datatables.net/buttons/2.4.1/css/buttons.dataTables.min.css">
<script src="https://cdn.datatables.net/buttons/2.4.1/js/dataTables.buttons.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/jszip/3.1.3/jszip.min.js"></script>
<script src="https://cdn.datatables.net/buttons/2.4.1/js/buttons.html5.min.js"></script>
<link rel="stylesheet" href="style.css">

```js libraries
// imports 
import * as Plot from "npm:@observablehq/plot";
import * as d3 from "npm:d3";
import CalHeatmap from "npm:cal-heatmap";
```

```js data
//data
const jsonData = FileAttachment("./data/project_summary.json").json();
```

```js create-timeline-data
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
```

```js aggregate-by-date
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

```js create-heatmap
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
    //label: 'D',
    radius: 2 },
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

```js timeline-table
function createTimelineEventsTable() {
  // Create a table container dynamically
  const container = document.getElementById("timeline-events-container");
  if (!container) {
    console.error("Table container for timeline not found!");
    return;
  }

  // Create the table element
  const table = document.createElement("table");
  table.id = "timeline-events-table";
  table.className = "display";

  // Clear the container and append the table
  container.innerHTML = "";
  container.appendChild(table);
  // Initialize DataTable directly with timelineEvents as the data source
  $("#timeline-events-table").DataTable({
    data: timelineEvents, // Directly use timelineEvents as the data source
    destroy: true, // Recreate the table each time
    columns: [
      { data: "name", title: "Event Name" },
      { 
        data: "date", 
        title: "Event Date",
      },
      { data: "type", title: "Event Type" }
    ],
    paging: true,
    searching: true,
    ordering: true,
    responsive: true,
    dom: "frtipB",
    buttons: [
      {
        extend: "csvHtml5",
        text: "Download CSV",
        title: "Timeline_Events",
        className: "btn btn-primary",
        exportOptions: {
          columns: ':visible', // Export visible columns only
          format: {
            header: function (data, columnIdx) {
              return $('#timeline-events-table thead th').eq(columnIdx).text().trim();
            }
          },
        },
      },
      {
        extend: "excelHtml5",
        text: "Download Excel",
        title: "Timeline_Events",
        className: "btn btn-success",
        exportOptions: {
          columns: ':visible', // Export visible columns only
          format: {
            header: function (data, columnIdx) {
              return $('#timeline-events-table thead th').eq(columnIdx).text().trim();
            }
          },
        },
      },
    ],
    order: [[1, "asc"]], 
    language: {
      search: "Search All: ", // Customize the label for the search box
    },
    initComplete: function () {
      // Optional: Add custom search inputs for each column
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
}

// Call the function to initialize the DataTable
createTimelineEventsTable();
```

```js calheatmap
function CalHeatmapComponent() {
	const intervals = {
		0: ['year', 'month', 4, 10, 20, 5, 2, 'MMMM YYYY'],
		1: ['year', 'week', 2, 5, 25, 0, 2, '[Week #]w YYYY'],
		2: ['year', 'day', 1, 10, 10, 1, 2, 'LL'],
		3: ['month', 'week', 10, 10, 10, 0, 0, '[week of] LL'],
		4: ['month', 'day', 10, 10, 10, 0, 2, 'LL'],
		5: ['day', 'hour', 12, 10, 10, 0, 2, 'LLL'],
		6: ['hour', 'minute', 9, 10, 10, 0, 2, 'LLLL'],
	};

	const schemes = {
		sequential: [
			'blues',
			'greens',
			'greys',
			'oranges',
			'purples',
			'reds',
			'bugn',
			'bupu',
			'gnbu',
			'orrd',
			'pubu',
			'pubugn',
			'purd',
			'rdpu',
			'ylgn',
			'ylgnbu',
			'ylorbr',
			'ylorrd',
			'cividis',
			'inferno',
			'magma',
			'plasma',
			'viridis',
			'cubehelix',
			'turbo',
			'warm',
			'cool',
		],
		diverging: [
			'brbg',
			'prgn',
			'piyg',
			'puor',
			'rdbu',
			'rdgy',
			'rdylbu',
			'rdylgn',
			'spectral',
			'burd',
			'buylrd',
		],
		cyclical: ['rainbow', 'sinebow'],
	};

	const dir = { asc: 'Left-to-Right', desc: 'Right-to-left' };

	const [selectedOption, setSelectedOption] = useState('Diverging');
	const [getCal, setCal] = useState(null);
	const [getScheme, setScheme] = useState('prgn');
	const [getDir, setDir] = useState('asc');
	const [getIntervalIndex, setIntervalIndex] = useState(2);

	const scales = {
		Diverging: {
			color: { type: 'diverging', scheme: getScheme, domain: [-10, 20] },
		},
		Threshold: {
			color: {
				type: 'threshold',
				scheme: getScheme,
				domain: [-10, -5, 0, 5, 10, 15, 20],
			},
		},
		Linear: {
			color: { type: 'linear', scheme: getScheme, domain: [-10, 20] },
		},
		Quantile: {
			color: { type: 'quantile', scheme: getScheme, domain: [-10, 20] },
		},
		Quantize: {
			color: { type: 'quantize', scheme: getScheme, domain: [-10, 20] },
		},
	};
	const options = {
		date: { start: new Date('2013-01-01') },
    data: {
      source: formattedEvents, // Use the aggregated timeline data
      x: 'date', 
      y: 'value',
    },
		scale: scales[selectedOption],
		itemSelector: '#cal-heatmap-index',
		domain: {
			type: intervals[getIntervalIndex][0],
			sort: getDir,
		},
		subDomain: {
			type: intervals[getIntervalIndex][1],
			radius: 2,
			sort: getDir,
		},
	};

	function paint(options) {
		options.domain.type = intervals[getIntervalIndex][0];
		options.domain.sort = getDir;

		options.subDomain.type = intervals[getIntervalIndex][1];
		options.subDomain.width = intervals[getIntervalIndex][3];
		options.subDomain.height = intervals[getIntervalIndex][4];
		options.subDomain.radius = intervals[getIntervalIndex][5];
		options.subDomain.gutter = intervals[getIntervalIndex][6];
		options.subDomain.sort = getDir;
		options.range = intervals[getIntervalIndex][2];

		getCal.paint(options, [
			[
				window.Tooltip,
				{
					text: function (date, value, dayjsDate) {
						return (
							(value ? value + '°C' : 'No data') +
							' on ' +
							dayjsDate.format(intervals[getIntervalIndex][7])
						);
					},
				},
			],
			[
				window.Legend,
				{
					itemSelector: '#cal-heatmap-index-legend',
					label: 'Seattle min. temp. (°C)',
				},
			],
		]);
	}

	function zoomIn() {
		if (getIntervalIndex >= Object.keys(intervals).length - 1) {
			return false;
		}

		if (getIntervalIndex === Object.keys(intervals).length - 2) {
			window.d3.select('#index-zoom-in').classed('disabled', true);
		}

		if (getIntervalIndex >= 0) {
			window.d3.select('#index-zoom-out').classed('disabled', false);
		}

		getCal.destroy().then(() => {
			setIntervalIndex(getIntervalIndex + 1);
			initCalendar();
		});
	}

	function zoomOut() {
		if (getIntervalIndex <= 0) {
			return false;
		}

		if (getIntervalIndex === 1) {
			window.d3.select('#index-zoom-out').classed('disabled', true);
		}

		if (getIntervalIndex != Object.keys(intervals).length - 2) {
			window.d3.select('#index-zoom-in').classed('disabled', false);
		}

		getCal.destroy().then(() => {
			setIntervalIndex(getIntervalIndex - 1);
			initCalendar();
		});
	}

	function selectScale(e) {
		setSelectedOption(e.target.value);
	}

	function selectScheme(e) {
		setScheme(e.target.value);
	}

	function selectDir(e) {
		setDir(e.target.value);
	}

	function initCalendar() {
		const cal = new window.CalHeatmap();
		setCal(cal);

		cal.on('resize', function (nw) {
			window.d3
				.select('#cal-heatmap-index-toolbar')
				.attr('style', `width: ${nw}px; opacity: 1`);

			window.d3
				.select('#cal-heatmap-index-footer')
				.attr('style', `width: ${nw}px; opacity: 1`);
		});
	}

	if (getCal === null) {
		initCalendar();
	} else {
		paint(options);
	}

	return (
		<div>
			<div id="cal-heatmap-index-toolbar">
				<div className="group-buttons">
					<a
						id="index-zoom-out"
						title="Zoom out"
						onClick={() => zoomOut()}
						className="button button--secondary button--sm padding-vert--xs padding-horiz--sm"
					>
						-
					</a>
					<a
						id="index-zoom-in"
						title="Zoom in"
						onClick={() => zoomIn()}
						className="button button--secondary button--sm padding-vert--xs padding-horiz--sm"
					>
						+
					</a>
				</div>
				<div></div>
				<div className="margin-right--md">
					<form>
						<select
							onChange={selectDir}
							value={getDir}
							className="button button--secondary button--sm"
						>
							<optgroup label="Choose a text direction">
								{Object.keys(dir).map(d => (
									<option key={d} value={d}>
										{dir[d]}
									</option>
								))}
							</optgroup>
						</select>
					</form>
				</div>
				<div className="margin-right--md">
					<form>
						<select
							onChange={selectScale}
							value={selectedOption}
							className="button button--secondary button--sm"
						>
							<optgroup label="Choose a scale type">
								{Object.keys(scales).map(o => (
									<option key={o} value={o}>
										{o}
									</option>
								))}
							</optgroup>
						</select>
					</form>
				</div>
				<div className="margin-right--md">
					<form>
						<select
							onChange={selectScheme}
							value={getScheme}
							className="button button--secondary button--sm"
						>
							<option disabled>Choose a color scheme</option>
							{Object.keys(schemes).map(title => (
								<optgroup
									key={title}
									label={title.toUpperCase()}
								>
									{schemes[title].map(o => (
										<option key={o} value={o}>
											{o}
										</option>
									))}
								</optgroup>
							))}
						</select>
					</form>
				</div>
				<div className="group-buttons">
					<a
						title="Previous"
						id="index-previous"
						onClick={() => getCal.previous()}
						className="button button--secondary button--secondary button--sm padding-vert--xs padding-horiz--sm"
					>
						‹
					</a>
					<a
						title="Next"
						id="index-next"
						onClick={() => getCal.next()}
						className="button button--secondary button--secondary button--sm padding-vert--xs padding-horiz--sm"
					>
						›
					</a>
				</div>
			</div>
			<div id="cal-heatmap-index"></div>
			<div id="cal-heatmap-index-footer">
				<small>Data may not be available for all timeframes</small>
				<div
					style={{ float: 'right' }}
					id="cal-heatmap-index-legend"
				></div>
			</div>
		</div>
	);
}

CalHeatmapComponent();
```

<div class ="card">
  <div class="card-title">
    <h1>Timeline Events</h1>
  </div>
  <div class="card-container">
    <div id="cal-heatmap-index"></div>
  </div>
</div>

- make scroll and have year at the bottom 
- add granulatory - but not minute 
- add a couple color options 


<div class="custom-collapse">
  <input type="checkbox" class="toggle-checkbox" id="collapse-toggle-timeline"> 
  <label for="collapse-toggle-timeline" class="collapse-title">
    <div class="card-title" id="timeline"><h1>Timeline Data</h1></div>
    <i class="expand-icon">+</i>
  </label>
  <div class="collapse-content">
    <p>This table includes the timeline data used to create the heatmap shown above. You can download the information using the buttons below.</p>
    <div id="timeline-events-container"></div> <!-- Placeholder for the table -->
  </div>
</div>



