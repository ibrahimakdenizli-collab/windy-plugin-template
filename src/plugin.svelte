<section class="plugin__content">
    <div class="plugin__title">Drone wind picker</div>
    <p class="mt-10 mb-15">
        Haritaya tıklayın, seçili katman ve irtifa için rüzgar verisini alın. Drone modelinize göre Go/No-Go uyarısı ve GPS güvenlik durumunu gösterir.
    </p>

    <div class="panel-row">
        <div class="panel-box">
            <label>Drone Model</label>
            <select bind:value={selectedDroneModelName}>
                {#each droneModels as model}
                    <option value={model.name}>{model.name} - max {model.maxWind} m/s, max {model.maxAltitude} m</option>
                {/each}
            </select>
            <div class="panel-note">Model limiti: <b>{selectedDroneModel.maxWind} m/s</b>, maksimum irtifa: <b>{selectedDroneModel.maxAltitude} m</b></div>
        </div>

        <div class="panel-box">
            <label>İrtifa (m)</label>
            <input type="range" min="0" max={selectedDroneModel.maxAltitude} step="10" bind:value={selectedAltitude} />
            <div>{selectedAltitude} m</div>
        </div>
    </div>

    <div class="panel-row stats-row">
        <div class="status-box {goNoGoClass}">{goNoGoText}</div>
        <div class="status-box {gpsStatusClass}">GPS durumu: {gpsSafety}</div>
    </div>

    {#if windData}
        <div class="info-box">
            <div><b>Koordinat:</b> {windData.lat.toFixed(4)}, {windData.lon.toFixed(4)}</div>
            <div><b>Seçili katman:</b> {activeLayer}</div>
            <div><b>Seçili irtifa:</b> {selectedAltitude} m</div>
            <div><b>Rüzgar hızı:</b> {windData.speed} {windData.unit}</div>
            <div><b>Rüzgar yönü:</b> {windData.dir}°</div>
            <div><b>Gust:</b> {windData.gust ?? 'Mevcut değil'}</div>
            <div><b>Geofence yarıçapı:</b> {geofenceRadius / 1000} km</div>
            {#if lastPickedAt}
                <div><b>Son veri:</b> {lastPickedAt.toLocaleString()}</div>
            {/if}
        </div>
    {:else}
        <div class="info-box">
            Haritaya tıklayarak seçili irtifa ve katman için rüzgar verisini alın.
        </div>
    {/if}

    {#if kpIndex !== null}
        <div class="kp-box">KP indeksi: <b>{kpIndex}</b> · {gpsSafety}</div>
    {/if}

    {#if lastError}
        <div class="error-box">Hata: {lastError}</div>
    {/if}

    <div class="actions">
        <button class="button" on:click={resetPicker}>Geofence ve değerleri temizle</button>
        <button class="button button--secondary" on:click={fetchKPIndex}>KP indeksini güncelle</button>
    </div>

    <small>Seçili katman: <b>{activeLayer}</b> · Seçili seviye: <b>{activeLevel}</b></small>
</section>

<script lang="ts">
    import { onDestroy, onMount } from 'svelte';
    import store from '@windy/store';
    import { map } from '@windy/map';
    import { singleclick } from '@windy/singleclick';
    import picker from '@windy/picker';
    import config from './pluginConfig';
    import type { LatLon } from '@windy/interfaces.d';

    interface DroneModel {
        name: string;
        maxWind: number;
        maxAltitude: number;
    }

    const { name } = config;
    const geofenceRadius = 5000;
    const droneModels: DroneModel[] = [
        { name: 'Light quadcopter', maxWind: 8, maxAltitude: 120 },
        { name: 'Pro drone', maxWind: 12, maxAltitude: 150 },
        { name: 'Heavy lift', maxWind: 16, maxAltitude: 200 },
    ];

    let selectedDroneModelName = droneModels[0].name;
    let selectedAltitude = droneModels[0].maxAltitude;

    let activeLayer = store.get('overlay') ?? 'wind';
    let activeLevel = store.get('level') ?? 'surface';
    let windData: {
        lat: number;
        lon: number;
        speed: number;
        dir: number;
        gust: number | null;
        unit: string;
    } | null = null;
    let lastPickedAt: Date | null = null;

    let kpIndex: number | null = null;
    let lastError: string | null = null;
    let geofence: L.Circle | null = null;
    let storeWatchInterval: number | null = null;
    let storeOnChange: (() => void) | null = null;

    $: selectedDroneModel = droneModels.find(model => model.name === selectedDroneModelName) ?? droneModels[0];
    $: if (selectedAltitude > selectedDroneModel.maxAltitude) {
        selectedAltitude = selectedDroneModel.maxAltitude;
    }

    $: goNoGoText = windData?.speed != null
        ? windData.speed > selectedDroneModel.maxWind
            ? 'No-Go: Rüzgar limiti aşıldı'
            : 'Go: Rüzgar limiti uygun'
        : 'Haritaya tıklayarak uçuşu kontrol edin';

    $: goNoGoClass = windData?.speed != null && windData.speed > selectedDroneModel.maxWind ? 'status-no-go' : 'status-go';
    $: gpsSafety = kpIndex === null ? 'KP bilinmiyor' : kpIndex <= 3 ? 'GPS Güvenli' : kpIndex <= 5 ? 'GPS Dikkat' : 'GPS Riskli';
    $: gpsStatusClass = kpIndex === null ? 'status-unknown' : kpIndex <= 3 ? 'status-go' : kpIndex <= 5 ? 'status-warning' : 'status-no-go';

    const updateSelectedOverlayAndLevel = () => {
        const overlay = store.get('overlay') ?? 'wind';
        const level = store.get('level') ?? 'surface';

        if (overlay !== activeLayer) {
            activeLayer = overlay;
        }
        if (level !== activeLevel) {
            activeLevel = level;
        }
    };

    const attachStoreWatcher = () => {
        if (typeof (store as any).on === 'function' && typeof (store as any).off === 'function') {
            (store as any).on('change', updateSelectedOverlayAndLevel);
            storeOnChange = () => {
                (store as any).off('change', updateSelectedOverlayAndLevel);
            };
        } else {
            storeWatchInterval = window.setInterval(updateSelectedOverlayAndLevel, 500);
        }
    };

    const drawGeofence = ({ lat, lon }: LatLon) => {
        if (geofence) {
            geofence.remove();
        }

        geofence = L.circle([lat, lon], {
            radius: geofenceRadius,
            color: '#faae2b',
            weight: 2,
            fillOpacity: 0.12,
        }).addTo(map);
    };

    const getPickerFunction = () => {
        if (typeof picker === 'function') {
            return picker;
        }

        return (picker as { get?: Function }).get;
    };

    const loadWindAt = async ({ lat, lon }: LatLon) => {
        activeLayer = store.get('overlay') ?? activeLayer;
        activeLevel = store.get('level') ?? activeLevel;
        lastError = null;

        const pickerFn = getPickerFunction();
        if (!pickerFn) {
            lastError = 'Picker API bulunamadı.';
            return;
        }

        try {
            const point = await pickerFn({
                lat,
                lon,
                overlay: activeLayer,
                level: selectedAltitude,
            });

            if (!point) {
                throw new Error('Harita üzerinden veri alınamadı.');
            }

            windData = {
                lat,
                lon,
                speed: point.wind ?? point.speed ?? 0,
                dir: point.dir ?? point.direction ?? 0,
                gust: point.gust ?? null,
                unit: point.unit ?? 'm/s',
            };
            lastPickedAt = new Date();
            drawGeofence({ lat, lon });
        } catch (error) {
            lastError = error instanceof Error ? error.message : String(error);
        }
    };

    const resetPicker = () => {
        windData = null;
        lastError = null;
        lastPickedAt = null;
        if (geofence) {
            geofence.remove();
            geofence = null;
        }
    };

    const fetchKPIndex = async () => {
        try {
            const response = await fetch('https://services.swpc.noaa.gov/products/noaa-planetary-k-index.json');
            if (!response.ok) {
                throw new Error(`KP verisi alınamadı: ${response.status}`);
            }
            const data = await response.json();
            const latest = Array.isArray(data) ? data[data.length - 1] : null;
            kpIndex = latest && typeof latest[1] === 'number' ? latest[1] : null;
        } catch (error) {
            kpIndex = null;
            lastError = error instanceof Error ? error.message : String(error);
        }
    };

    const onMapClick = (location: LatLon) => {
        if (!location || typeof location.lat !== 'number' || typeof location.lon !== 'number') {
            return;
        }

        loadWindAt(location);
    };

    onMount(() => {
        updateSelectedOverlayAndLevel();
        attachStoreWatcher();
        fetchKPIndex();
        singleclick.on(name, onMapClick);
    });

    onDestroy(() => {
        singleclick.off(name, onMapClick);

        if (storeWatchInterval !== null) {
            window.clearInterval(storeWatchInterval);
            storeWatchInterval = null;
        }

        if (storeOnChange) {
            storeOnChange();
            storeOnChange = null;
        }

        if (geofence) {
            geofence.remove();
            geofence = null;
        }
    });
</script>

<style>
    .plugin__content {
        display: flex;
        flex-direction: column;
        gap: 12px;
    }

    .plugin__title {
        font-size: 1.1rem;
        font-weight: 700;
    }

    .panel-row {
        display: flex;
        gap: 12px;
        flex-wrap: wrap;
    }

    .panel-box {
        flex: 1;
        min-width: 200px;
        padding: 12px;
        border: 1px solid rgba(255, 255, 255, 0.12);
        border-radius: 10px;
        background: rgba(255, 255, 255, 0.04);
    }

    .panel-box label {
        display: block;
        margin-bottom: 8px;
        font-weight: 600;
    }

    .panel-note {
        margin-top: 8px;
        color: #dcdcdc;
        font-size: 0.9rem;
    }

    .stats-row {
        align-items: stretch;
    }

    .status-box {
        flex: 1;
        padding: 14px;
        border-radius: 10px;
        text-align: center;
        font-weight: 700;
        min-width: 170px;
    }

    .status-go {
        background: rgba(46, 204, 113, 0.14);
        border: 1px solid rgba(46, 204, 113, 0.3);
        color: #d4f1dc;
    }

    .status-warning {
        background: rgba(241, 196, 15, 0.14);
        border: 1px solid rgba(241, 196, 15, 0.3);
        color: #f7e7b0;
    }

    .status-no-go {
        background: rgba(231, 76, 60, 0.14);
        border: 1px solid rgba(231, 76, 60, 0.3);
        color: #f4c6c4;
    }

    .status-unknown {
        background: rgba(149, 165, 166, 0.14);
        border: 1px solid rgba(149, 165, 166, 0.3);
        color: #bdc3c7;
    }

    .info-box,
    .kp-box {
        border: 1px solid rgba(255, 255, 255, 0.12);
        border-radius: 10px;
        padding: 12px;
        background: rgba(255, 255, 255, 0.04);
    }

    .error-box {
        border: 1px solid rgba(255, 80, 80, 0.7);
        border-radius: 10px;
        padding: 10px 12px;
        color: #ffb3b3;
        background: rgba(255, 80, 80, 0.08);
    }

    .actions {
        display: flex;
        gap: 8px;
        flex-wrap: wrap;
    }

    .button {
        border: 1px solid rgba(255, 255, 255, 0.12);
        border-radius: 8px;
        background: transparent;
        color: #fff;
        padding: 8px 12px;
        cursor: pointer;
    }

    .button:hover {
        background: rgba(255, 255, 255, 0.08);
    }

    .button--secondary {
        border-color: rgba(255, 255, 255, 0.08);
    }

    small {
        color: #dcdcdc;
    }
</style>

