<script>
    import { onMount } from "svelte";
    import mapboxgl from "mapbox-gl";
    import * as d3 from "d3";
    import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";

    let map;
    let stations = [];
    let trips = [];
    let mapViewChanged = 0;

    let departures;
    let arrivals;

    let timeFilter = -1;
    $: timeFilterLabel = new Date(0, 0, 0, 0, timeFilter).toLocaleString("en", { timeStyle: "short" });

    const departuresByMinute = Array.from({ length: 1440 }, () => []);
    const arrivalsByMinute = Array.from({ length: 1440 }, () => []);

    mapboxgl.accessToken = "pk.eyJ1IjoiYW1pcmFyYXYiLCJhIjoiY21wMDhsYXFkMWhlbTJyb2c3d3RqZWFvOCJ9.TFezeIWMyWO4ze8H4dtRgA";

    const bikeLaneStyle = {
        "line-color": "#32a852",
        "line-width": 3,
        "line-opacity": 0.4
    };

    $: radiusScale = d3.scaleSqrt()
        .domain([0, d3.max(stations, d => d.totalTraffic) || 0])
        .range(timeFilter === -1 ? [0, 25] : [3, 50]);

    let stationFlow = d3.scaleQuantize()
        .domain([0, 1])
        .range([0, 0.5, 1]);

    const urlBase = 'https://api.mapbox.com/isochrone/v1/mapbox/';
    const profile = 'cycling';
    const minutes = [5, 10, 15, 20];
    const contourColors = [
        "03045e",
        "0077b6",
        "00b4d8",
        "90e0ef"
    ];
    let isochrone = null;
    let selectedStation = null;

    $: if (selectedStation) {
        getIso(+selectedStation.Long, +selectedStation.Lat);
    } else {
        isochrone = null;
    }

    async function getIso(lon, lat) {
        const base = `${urlBase}${profile}/${lon},${lat}`;
        const params = new URLSearchParams({
            contours_minutes: minutes.join(','),
            contours_colors: contourColors.join(','),
            polygons: 'true',
            access_token: mapboxgl.accessToken
        });
        const url = `${base}?${params.toString()}`;
        const query = await fetch(url, { method: 'GET' });
        isochrone = await query.json();
    }

    function geoJSONPolygonToPath(feature) {
        const path = d3.path();
        const rings = feature.geometry.coordinates;
        for (const ring of rings) {
            for (let i = 0; i < ring.length; i++) {
                const [lng, lat] = ring[i];
                const { x, y } = map.project([lng, lat]);
                if (i === 0) path.moveTo(x, y);
                else path.lineTo(x, y);
            }
            path.closePath();
        }
        return path.toString();
    }

    function getCoords(station) {
        let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
        let { x, y } = map.project(point);
        return { cx: x, cy: y };
    }

    function minutesSinceMidnight(date) {
        return date.getHours() * 60 + date.getMinutes();
    }

    function filterByMinute(tripsByMinute, minute) {
        let minMinute = (minute - 60 + 1440) % 1440;
        let maxMinute = (minute + 60) % 1440;

        if (minMinute > maxMinute) {
            let beforeMidnight = tripsByMinute.slice(minMinute);
            let afterMidnight = tripsByMinute.slice(0, maxMinute);
            return beforeMidnight.concat(afterMidnight).flat();
        } else {
            return tripsByMinute.slice(minMinute, maxMinute).flat();
        }
    }

    $: filteredDepartures = timeFilter === -1 ? departures : d3.rollup(filterByMinute(departuresByMinute, timeFilter), v => v.length, d => d.start_station_id);
    $: filteredArrivals = timeFilter === -1 ? arrivals : d3.rollup(filterByMinute(arrivalsByMinute, timeFilter), v => v.length, d => d.end_station_id);

    $: filteredStations = stations.map(station => {
        let cloned = { ...station };
        let id = cloned.Number;
        cloned.arrivals = filteredArrivals.get(id) ?? 0;
        cloned.departures = filteredDepartures.get(id) ?? 0;
        cloned.totalTraffic = cloned.arrivals + cloned.departures;
        return cloned;
    });

    async function initMap() {
        map = new mapboxgl.Map({
            container: "map",
            style: "mapbox://styles/mapbox/streets-v12",
            center: [-71.09415, 42.36027],
            zoom: 12,
            minZoom: 5,
            maxZoom: 18
        });

        await new Promise(resolve => map.on("load", resolve));

        map.addSource("boston_route", {
            type: "geojson",
            data: "./Existing_Bike_Network_2022.geojson"
        });

        map.addSource("cambridge_route", {
            type: "geojson",
            data: "https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson"
        });

        map.addLayer({ id: "boston-bike-lanes", type: "line", source: "boston_route", paint: bikeLaneStyle });
        map.addLayer({ id: "cambridge-bike-lanes", type: "line", source: "cambridge_route", paint: bikeLaneStyle });

        map.on("move", () => mapViewChanged++);
    }

    onMount(async () => {
        await initMap();

        const stationData = await d3.csv("https://vis-society.github.io/labs/9/data/bluebikes-stations.csv");

        trips = await d3.csv("https://vis-society.github.io/labs/9/data/bluebikes-traffic-2024-03.csv").then(trips => {
            for (let trip of trips) {
                trip.started_at = new Date(trip.started_at);
                trip.ended_at = new Date(trip.ended_at);

                let startedMinutes = minutesSinceMidnight(trip.started_at);
                departuresByMinute[startedMinutes].push(trip);

                let endedMinutes = minutesSinceMidnight(trip.ended_at);
                arrivalsByMinute[endedMinutes].push(trip);
            }
            return trips;
        });

        departures = d3.rollup(trips, v => v.length, d => d.start_station_id);
        arrivals = d3.rollup(trips, v => v.length, d => d.end_station_id);

        stations = stationData.map(station => {
            let id = station.Number;
            station.arrivals = arrivals.get(id) ?? 0;
            station.departures = departures.get(id) ?? 0;
            station.totalTraffic = station.arrivals + station.departures;
            return station;
        });
    });
</script>

<header>
    <h1>Bikewatching</h1>
    <label>
        Filter by time:
        <input type="range" min="-1" max="1440" bind:value={timeFilter} />
        {#if timeFilter !== -1}
            <time style="display: block">{timeFilterLabel}</time>
        {:else}
            <em style="display: block">(any time)</em>
        {/if}
    </label>
</header>
<p>
    A map of Bluebikes bike-share traffic in the Boston area, based on trips from March 2024. Each circle is a station. Its size reflects the total number of trips starting or ending there, and its color shows whether departures (blue) or arrivals (orange) are more common, with purple meaning roughly balanced. Green lines mark existing bike lanes in Boston and Cambridge.
</p>
<p>
    Drag the slider to show only trips within an hour of the chosen time of day. Click a station to see the estimated area reachable from it by bike in 5, 10, 15 and 20 minutes, and click it again to clear.
</p>

<div id="map">
    <svg>
        {#key mapViewChanged}
            {#if isochrone}
                {#each isochrone.features as feature}
                    <path
                        d={geoJSONPolygonToPath(feature)}
                        fill={feature.properties.fillColor}
                        fill-opacity="0.2"
                        stroke="#000"
                        stroke-opacity="0.5"
                        stroke-width="1"
                    >
                        <title>{feature.properties.contour} minutes of biking</title>
                    </path>
                {/each}
            {/if}
            {#each filteredStations as station}
                <circle
                    {...getCoords(station)}
                    r={radiusScale(station.totalTraffic)}
                    style="--departure-ratio: {stationFlow(station.departures / station.totalTraffic)}"
                    class={station?.Number === selectedStation?.Number ? "selected" : ""}
                    on:mousedown={() => {
                        selectedStation = selectedStation?.Number !== station?.Number ? station : null;
                    }}
                    fill-opacity="0.6"
                    stroke="white"
                >
                    <title>{station.totalTraffic} trips ({station.departures} departures, {station.arrivals} arrivals)</title>
                </circle>
            {/each}
        {/key}
    </svg>
</div>

<div class="legend">
    <div style="--departure-ratio: 1">More departures</div>
    <div style="--departure-ratio: 0.5">Balanced</div>
    <div style="--departure-ratio: 0">More arrivals</div>
</div>

<p>
    Data Sources:
    <a href="https://bluebikes.com/system-data">Bluebikes system data</a> (stations and March 2024 trips),
    <a href="https://data.boston.gov/dataset/existing-bike-network-2022">City of Boston Existing Bike Network 2022</a>,
    <a href="https://github.com/cambridgegis/cambridgegis_data">Cambridge GIS Bike Facilities</a>.
</p>

<style>
    @import url("$lib/global.css");

    header {
        display: flex;
        gap: 1em;
        align-items: baseline;
    }

    header label {
        margin-left: auto;
    }

    #map {
        flex: 1;
        position: relative;
    }

    #map svg {
        position: absolute;
        z-index: 1;
        width: 100%;
        height: 100%;
        pointer-events: none;
    }

    circle, .legend > div {
        --color-departures: steelblue;
        --color-arrivals: darkorange;
        --color: color-mix(
            in oklch,
            var(--color-departures) calc(100% * var(--departure-ratio)),
            var(--color-arrivals)
        );
    }

    circle {
        pointer-events: auto;
        fill: var(--color);
        transition: opacity 0.2s ease;
    }

    path {
        pointer-events: auto;
    }

    #map svg:has(circle.selected) circle:not(.selected) {
        opacity: 0.3;
    }

    .legend {
        display: flex;
        gap: 1px;
        margin-block: 1em;
    }

    .legend > div {
        flex: 1;
        background-color: var(--color);
        color: white;
        text-align: center;
        padding: 0.5em 1em;
    }
</style>
