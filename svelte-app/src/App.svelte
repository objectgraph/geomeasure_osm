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

	const measurementControl = L.Control.extend({
		options: {
			position: "topright",
		},

		onAdd: function (map) {
			const container = L.DomUtil.create("div", "measurement-control");

			// Toggle Button
			const toggleButton = L.DomUtil.create(
				"button",
				"toggle-button",
				container
			);
			toggleButton.innerHTML = "Toggle Measurement";
			toggleButton.onclick = () => {
				modeChange();
				// Additional logic to update UI
			};

			// Distance Label
			const distanceLabel = L.DomUtil.create(
				"div",
				"distance-label",
				container
			);
			distanceLabel.id = "distance";
			distanceLabel.innerHTML = "Distance: 0 meters";

			// Area Label
			const areaLabel = L.DomUtil.create("div", "area-label", container);
			areaLabel.id = "area";
			areaLabel.innerHTML = "Area: 0 square meters";
			L.DomEvent.on(container, "click", L.DomEvent.stopPropagation);
			return container;
		},
	});

	function formatDistance(value) {
		switch (distanceUnit) {
			case "kilometers":
				return `${(value / 1000).toFixed(2)} kilometers`;
			case "miles":
				return `${(value * 0.000621371).toFixed(2)} miles`;
			default:
				return `${value.toFixed(2)} meters`;
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
		} else {
			polygon.setLatLngs(latlngs);
		}
	}

	function formatArea(value) {
		switch (areaUnit) {
			case "squareKilometers":
				return `${(value / 1e6).toFixed(2)} square kilometers`;
			case "acres":
				return `${(value * 0.000247105).toFixed(2)} acres`;
			default:
				return `${value.toFixed(2)} square meters`;
		}
	}

	const modeChange = () => {
		mode = mode === "distance" ? "area" : "distance";
		polygon.remove();
		polygon = L.polygon([], { color: "red" });
		polyline.remove();
		polyline = L.polyline([], { color: "blue" });
		markers.clearLayers();
		document.getElementById("distance").innerText = "Distance: 0 meters";
		document.getElementById("area").innerText = "Area: 0 square meters";
	};

	onMount(() => {
		const map = L.map("map").setView([51.505, -0.09], 13);
		markers.addTo(map);

		const control = new measurementControl();
		control.addTo(map);

		L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png").addTo(
			map
		);
		let distance = 0;
		let dragging = false;

		// Handle mouseup event
		map.on("mouseup", function () {
			if (dragging) {
				dragging = false;
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
			const marker = L.circleMarker(e.latlng, {
				color: "blue",
				fillColor: "#30f",
				fillOpacity: 0.5,
				radius: 5,
				draggable: true,
			});

			// Handle mousedown event
			marker.on("mousedown", function (e) {
				dragging = true;
				currentMarker = marker;
				map.dragging.disable();
				map.on("mousemove", onMouseMove);
			});

			marker.addTo(markers);
			if (mode === "distance") {
				polyline.addTo(map);
				polyline.addLatLng(e.latlng);
				if (polyline.getLatLngs().length > 1) {
					distance += e.latlng.distanceTo(
						polyline.getLatLngs()[polyline.getLatLngs().length - 2]
					);
					document.getElementById(
						"distance"
					).innerText = `Distance: ${formatDistance(distance)}`;
				}
				polyline
					.bindTooltip(`Distance: ${formatDistance(distance)}`)
					.openTooltip();
			} else {
				polygon.addLatLng(e.latlng);
				polygon.addTo(map);
				if (polygon.getLatLngs()[0].length > 2) {
					const coordinates = polygon
						.getLatLngs()[0]
						.map((latlng) => [latlng.lng, latlng.lat]);
					coordinates.push(coordinates[0]); // Close the polygon
					const turfPolygon = turf.polygon([coordinates]);
					const area = turf.area(turfPolygon);
					polygon
						.bindTooltip(`Area: ${formatArea(area)}`)
						.openTooltip();
					document.getElementById(
						"area"
					).innerText = `Area: ${formatArea(area)}`;
				}
			}
		});
	});
</script>

<main>
	<select bind:value={distanceUnit}>
		<option value="meters">Meters</option>
		<option value="kilometers">Kilometers</option>
		<option value="miles">Miles</option>
	</select>

	<select bind:value={areaUnit}>
		<option value="squareMeters">Square Meters</option>
		<option value="squareKilometers">Square Kilometers</option>
		<option value="acres">Acres</option>
	</select>

	<div id="map" style="height: 500px;" />
	<div id="distance">Distance: 0 meters</div>
	<div id="area">Area: 0 square meters</div>
	<button on:click={modeChange}>
		Toggle to {mode === "distance" ? "Area" : "Distance"} Measurement
	</button>
</main>

<style>
	main {
		text-align: center;
		padding: 1em;
		max-width: 240px;
		margin: 0 auto;
	}
	@media (min-width: 640px) {
		main {
			max-width: none;
		}
	}
</style>
