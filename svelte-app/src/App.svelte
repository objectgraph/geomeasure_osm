<script>
	import L from "leaflet";
	import { onMount } from "svelte";
	import * as turf from "@turf/turf";
	let mode = "distance"; // Default mode
	let polygon = L.polygon([], { color: "red" });
	let polyline = L.polyline([], { color: "blue" });
	const markers = L.layerGroup();
	let distanceUnit = "meters";
	let areaUnit = "squareMeters";
	let currentMarker = null;
	let justReleased = false;
	let distanceUnitDropdown;
	let areaUnitDropdown;
	let undoStack = [];
	let redoStack = [];

	function undo() {
		if (undoStack.length > 0) {
			const lastMarker = undoStack.pop();
			redoStack.push(lastMarker);
			markers.removeLayer(lastMarker);
			update();
		}
	}

	function redo() {
		if (redoStack.length > 0) {
			const lastMarker = redoStack.pop();
			undoStack.push(lastMarker);
			markers.addLayer(lastMarker);
			update();
		}
	}

	const measurementControl = L.Control.extend({
		options: {
			position: "topright",
		},

		onAdd: function (map) {
			const container = L.DomUtil.create("div", "measurement-control");
			// Project Label
			const projectLabel = L.DomUtil.create("label", "project-label", container);
			projectLabel.innerHTML = "GeoMeasure";

			// Distance Unit Dropdown
			distanceUnitDropdown = L.DomUtil.create(
				"select",
				"distance-unit-dropdown",
				container
			);
			["meters", "kilometers", "miles"].forEach((unit) => {
				const option = L.DomUtil.create(
					"option",
					"",
					distanceUnitDropdown
				);
				option.value = unit;
				option.text = unit.charAt(0).toUpperCase() + unit.slice(1);
				if (unit === distanceUnit) {
					option.selected = true;
				}
			});
			distanceUnitDropdown.onchange = (e) => {
				distanceUnit = e.target.value;
				update();
			};

			// Area Unit Dropdown
			areaUnitDropdown = L.DomUtil.create(
				"select",
				"area-unit-dropdown",
				container
			);
			["squareMeters", "squareKilometers", "acres"].forEach((unit) => {
				const option = L.DomUtil.create("option", "", areaUnitDropdown);
				option.value = unit;
				option.text = unit.charAt(0).toUpperCase() + unit.slice(1);
				if (unit === areaUnit) {
					option.selected = true;
				}
			});
			areaUnitDropdown.onchange = (e) => {
				areaUnit = e.target.value;
				update();
			};

			// Initially hide the dropdowns
			distanceUnitDropdown.style.display = "none";
			areaUnitDropdown.style.display = "none";

			// Show the correct dropdown based on the mode
			if (mode === "distance") {
				distanceUnitDropdown.style.display = "block";
			} else {
				areaUnitDropdown.style.display = "block";
			}

			// Toggle Button
			const toggleButton = L.DomUtil.create(
				"button",
				"mode-button",
				container
			);
			toggleButton.innerHTML = "Mode";
			toggleButton.onclick = () => {
				modeChange();
			};

			const clearButton = L.DomUtil.create(
				"button",
				"clear-button",
				container
			);
			clearButton.innerHTML = "Clear";
			clearButton.onclick = () => {
				clear();
			};

			const undoButton = L.DomUtil.create(
				"button",
				"clear-button",
				container
			);
			undoButton.innerHTML = "Undo";
			L.DomEvent.on(undoButton, "click", (e) => {
				undo();
			});

			const redoButton = L.DomUtil.create(
				"button",
				"clear-button",
				container
			);
			redoButton.innerHTML = "Redo";
			L.DomEvent.on(redoButton, "click", (e) => {
				redo();
			});

			// Distance Label
			const outputLabel = L.DomUtil.create(
				"div",
				"output-label",
				container
			);
			outputLabel.id = "output";
			outputLabel.innerHTML = "Distance: 0 m";

			// Search Input
			const searchInput = L.DomUtil.create(
				"input",
				"search-input",
				container
			);
			searchInput.type = "text";
			searchInput.placeholder = "Enter an address";

			L.DomEvent.on(searchInput, "keydown", (e) => {
				if (e.key === "Enter") {
					e.preventDefault();
					searchAddress();
				}
			});

			// Search Button
			const searchButton = L.DomUtil.create(
				"button",
				"search-button",
				container
			);
			searchButton.innerHTML = "Search";
			searchButton.onclick = () => {
				searchAddress();
			};

			function searchAddress() {
				const address = searchInput.value;
				fetch(
					`https://nominatim.openstreetmap.org/search?format=json&q=${address}`
				)
					.then((response) => response.json())
					.then((data) => {
						if (data.length > 0) {
							const { boundingbox } = data[0];
							const southWest = L.latLng(
								boundingbox[0],
								boundingbox[2]
							);
							const northEast = L.latLng(
								boundingbox[1],
								boundingbox[3]
							);
							const bounds = L.latLngBounds(southWest, northEast);
							map.fitBounds(bounds);
						} else {
							alert("Address not found");
						}
					});
			}

			L.DomEvent.disableClickPropagation(container);
			return container;
		},
	});

	function formatDistance(value) {
		switch (distanceUnit) {
			case "kilometers":
				return `${(value / 1000).toFixed(2)} km`;
			case "miles":
				return `${(value * 0.000621371).toFixed(2)} mi`;
			default:
				return `${value.toFixed(2)} m`;
		}
	}

	function formatArea(value) {
		switch (areaUnit) {
			case "squareKilometers":
				return `${(value / 1e6).toFixed(2)} km<sup>2</sup>`;
			case "acres":
				return `${(value * 0.000247105).toFixed(2)} ac`;
			default:
				return `${value.toFixed(2)} m<sup>2</sup>`;
		}
	}

	function update() {
		// Get all the latlngs from the markers layer group
		const latlngs = [];
		markers.eachLayer((marker) => {
			latlngs.push(marker.getLatLng());
		});

		// Update the polyline with the new latlngs
		if (mode == "distance") {
			polyline.setLatLngs(latlngs);
			let distance = 0;
			if (polyline.getLatLngs().length > 1) {
				const coordinates = polyline
					.getLatLngs()
					.map((latlng) => [latlng.lng, latlng.lat]);

				// Create a line string from the coordinates
				const lineString = turf.lineString(coordinates);

				// Calculate the length of the line string
				distance = turf.length(lineString, { units: "meters" });
			} else {
				distance = 0;
			}
			document.getElementById(
				"output"
			).innerHTML = `Distance: ${formatDistance(distance)}`;
			polyline
				.bindTooltip(`Distance: ${formatDistance(distance)}`)
				.openTooltip();
		} else {
			polygon.setLatLngs(latlngs);
			let area = 0;
			if (polygon.getLatLngs()[0].length > 2) {
				const coordinates = polygon
					.getLatLngs()[0]
					.map((latlng) => [latlng.lng, latlng.lat]);
				coordinates.push(coordinates[0]); // Close the polygon
				const turfPolygon = turf.polygon([coordinates]);
				area = turf.area(turfPolygon);
			} else {
				area = 0;
			}

			polygon.bindTooltip(`Area: ${formatArea(area)}`).openTooltip();
			document.getElementById("output").innerHTML = `Area: ${formatArea(
				area
			)}`;
		}
	}

	function clear() {
		polygon.remove();
		polygon = L.polygon([], { color: "red" });
		polyline.remove();
		polyline = L.polyline([], { color: "blue" });
		markers.clearLayers();
		if (mode == "distance") {
			document.getElementById("output").innerHTML = "Distance: 0 m";
		} else {
			document.getElementById("output").innerHTML =
				"Area: 0 m<sup>2</sup>";
		}
	}

	function modeChange() {
		mode = mode === "distance" ? "area" : "distance";
		clear();

		// Hide/show the dropdowns
		if (mode === "distance") {
			distanceUnitDropdown.style.display = "block";
			areaUnitDropdown.style.display = "none";
		} else {
			distanceUnitDropdown.style.display = "none";
			areaUnitDropdown.style.display = "block";
		}
	}

	onMount(() => {
		const osmLayer = L.tileLayer(
			"https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png",
			{
				maxZoom: 19,
				attribution:
					'Map data: &copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a>',
			}
		);

		const openTopoLayer = L.tileLayer(
			"https://{s}.tile.opentopomap.org/{z}/{x}/{y}.png",
			{
				maxZoom: 17,
				attribution:
					'Map data: &copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a>, ' +
					'Tiles courtesy of <a href="https://opentopomap.org/">OpenTopoMap</a>',
			}
		);

		const baseLayers = {
			OSM: osmLayer,
			OpenTopoMap: openTopoLayer,
		};
		const map = L.map("map", {
			center: [51.505, -0.09],
			zoom: 13,
			layers: [osmLayer], // Set the default layer
		});
		markers.addTo(map);

		const control = new measurementControl();
		control.addTo(map);

		L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png").addTo(
			map
		);

		L.control.layers(baseLayers).addTo(map);
		let dragging = false;

		// Handle mouseup touchend event
		map.on("mouseup touchend", function () {
			if (dragging) {
				dragging = false;
				justReleased = true; // Set justReleased to true
				map.off("mousemove", onMouseMove);
				map.dragging.enable(); // Re-enable map dragging
				update();
			}
		});

		// Handle mouse move event
		function onMouseMove(e) {
			if (dragging) {
				currentMarker.setLatLng(e.latlng);
			}
		}

		map.on("click", (e) => {
			if (justReleased) {
				justReleased = false; // Reset justReleased
				return; // Skip the rest of the logic
			}
			const marker = L.circleMarker(e.latlng, {
				color: "blue",
				fillColor: "#30f",
				fillOpacity: 0.5,
				radius: 5,
				draggable: true,
			});

			// Handle mousedown touchstart event
			marker.on("mousedown touchstart", function (e) {
				justReleased = false; // Reset justReleased
				dragging = true;
				currentMarker = marker;
				map.dragging.disable();
				map.on("mousemove touchmove", onMouseMove);
			});

			marker.addTo(markers);
			undoStack.push(marker);
			redoStack = []; // Clear redo stack when new action is made

			if (mode === "distance") {
				polyline.addTo(map);
				polyline.addLatLng(e.latlng);
			} else {
				polygon.addLatLng(e.latlng);
				polygon.addTo(map);
			}

			update();
		});
	});
</script>

<main>
	<div id="map" />
</main>

<style>
	main {
		position: absolute;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		padding: 0;
		margin: 0;
	}

	#map {
		width: 100%;
		height: 100%;
	}
	@media (min-width: 640px) {
		main {
			max-width: none;
		}
	}
</style>
