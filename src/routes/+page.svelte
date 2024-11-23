<script lang="ts">
	import * as L from 'leaflet';
	// If you're playing with this in the Svelte REPL, import the CSS using the
	// syntax in svelte:head instead. For normal development, this is better.
	import 'leaflet/dist/leaflet.css';

	let map = $state<L.Map>();
	let lat = $state(13.7563);
	let lng = $state(100.5018);
	let flyToMarker = $state<boolean>(true);

	let markers = $state<L.Marker[]>([]);

	const initMarkers = [
		[13.7563, 100.5018],
		[13.75458, 100.49042],
		[13.69611, 100.497608],
		[14.002367, 99.99095]
	];

	const bangkokLatLng = L.latLng(13.7563, 100.5018);
	function createMap(container: HTMLDivElement) {
		let m = L.map(container).setView(bangkokLatLng, 13);
		L.tileLayer('https://{s}.basemaps.cartocdn.com/rastertiles/voyager/{z}/{x}/{y}{r}.png', {
			attribution: `GPS demo`,
			maxZoom: 16
		}).addTo(m);

		return m;
	}

	function mapAction(container: HTMLDivElement) {
		map = createMap(container);

		initMarkers.forEach((latlng) => {
			createMarker(latlng[0], latlng[1]);
		});

		map.on('click', (e) => {
			createMarker(e.latlng.lat, e.latlng.lng);
		});

		return {
			destroy: () => {
				map?.remove();
			}
		};
	}

	function resizeMap() {
		if (map) {
			map.invalidateSize();
		}
	}

	// Create a popup with a Svelte component inside it and handle removal when the popup is torn down.
	// `createFn` will be called whenever the popup is being created, and should create and return the component.
	function bindPopup(marker: L.Marker, createFn: (container: HTMLDivElement) => HTMLDivElement) {
		let popupComponent: any;
		marker.bindPopup(() => {
			let container = L.DomUtil.create('div');
			popupComponent = createFn(container);
			return container;
		});
	}

	function createMarker(lat: number, lng: number) {
		fetch(`https://nominatim.openstreetmap.org/reverse?lat=${lat}&lon=${lng}&format=json`, {
			headers: {
				'Accept-Language': 'th-th'
			}
		})
			.then((response) => response.json())
			.then((data) => {
				if (!map) return;
				let marker = L.marker([lat, lng], {
					alt: data.name || 'UNKNOWN'
				}).addTo(map);
				bindPopup(marker, (container) => {
					const button = document.createElement('button');
					button.textContent = 'Remove';
					button.className = 'block ml-auto pt-2 text-red-700 hover:underline';
					button.onclick = () => {
						marker.remove();
						markers = markers.filter((m) => m !== marker);
					};
					container.innerHTML = `
        <div>
          <p class="text-lg font-bold">${data.name || 'UNKNOWN'}</p>
          <p>Latitude: ${marker.getLatLng()}</p>
        </div>
      `;
					container.appendChild(button);
					return container;
				});

				markers.push(marker);
			});
	}

	function flyTo(marker: L.Marker) {
		if (!map) return;

		map.flyTo(marker.getLatLng(), 15, {
			duration: 1.5,
			animate: flyToMarker
		});
	}
</script>

<svelte:window on:resize={resizeMap} />

<svelte:head>
	<!-- In the REPL you need to do this. In a normal Svelte app, use a CSS Rollup plugin and import it from the leaflet package. -->
	<link
		rel="stylesheet"
		href="https://unpkg.com/leaflet@1.6.0/dist/leaflet.css"
		integrity="sha512-xwE/Az9zrjBIphAcBb3F6JVqxf46+CDLwfLMHloNu6KEQCAWi6HcDUbeOfBIptF7tcCzusKFjFw2yuvEpDL9wQ=="
		crossorigin=""
	/>
</svelte:head>

<div class="flex h-[100svh] flex-col">
	<div class="flex w-full items-center">
		<div class="mr-10 flex-1">
			<div class="w-[50vw] overflow-x-scroll">
				{#if markers.length === 0}
					<span>No markers, Click on the map to add some</span>
				{/if}
				{#each markers as marker}
					<button
						class="rouneded inline-flex border bg-white px-2 py-1 text-blue-600"
						onclick={() => flyTo(marker)}
					>
						<span class="p-0 text-lg font-bold">{marker.options.alt}</span>
					</button>
				{/each}
			</div>
		</div>
		<!-- lat, lng -->
		<input type="checkbox" bind:checked={flyToMarker} />
		<label for="flyToMarker" class="ml-2 mr-10">Fly to Marker</label>
		<label for="lat">Lat</label>
		<input class="block w-36 py-1" type="number" bind:value={lat} step="0.0001" />
		<label for="lng" class="ml-2">Lng</label>
		<input class="block w-36 py-1" type="number" bind:value={lng} step="0.0001" />
		<button
			class=" ml-2 rounded bg-blue-500 px-4 py-2 font-bold text-white hover:bg-blue-700"
			onclick={() => createMarker(lat, lng)}>Create Marker</button
		>
	</div>
	<div class="flex-1" use:mapAction></div>
</div>
