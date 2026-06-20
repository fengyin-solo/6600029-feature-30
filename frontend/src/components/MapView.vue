<script setup lang="ts">
import { onMounted, onUnmounted, ref, watch, nextTick } from 'vue';
import L from 'leaflet';
import 'leaflet/dist/leaflet.css';
import { useDroneStore } from '../store/drone';

const store = useDroneStore();
const mapContainer = ref<HTMLElement>();
let map: L.Map | null = null;
let waypointLayer: L.LayerGroup | null = null;
let routeLayer: L.Polyline | null = null;
let zoneLayer: L.LayerGroup | null = null;
let droneMarker: L.CircleMarker | null = null;
let segmentLabelLayer: L.LayerGroup | null = null;

const addMode = ref(false);
const currentZoom = ref(12);

const ACTION_META: Record<string, { icon: string; color: string; label: string }> = {
  hover: { icon: '⏸', color: '#f59e0b', label: '悬停' },
  photo: { icon: '📷', color: '#3b82f6', label: '拍照' },
  video: { icon: '🎬', color: '#8b5cf6', label: '录像' },
  none: { icon: '•', color: '#6b7280', label: '无动作' },
};

function getZoomDensity(): 'high' | 'medium' | 'low' {
  if (currentZoom.value >= 15) return 'high';
  if (currentZoom.value >= 12) return 'medium';
  return 'low';
}

function initMap() {
  if (!mapContainer.value || map) return;
  map = L.map(mapContainer.value).setView(store.mapCenter, 12);
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: '© OpenStreetMap',
    maxZoom: 18,
  }).addTo(map);

  waypointLayer = L.layerGroup().addTo(map);
  zoneLayer = L.layerGroup().addTo(map);
  segmentLabelLayer = L.layerGroup().addTo(map);

  map.on('click', (e: L.LeafletMouseEvent) => {
    if (addMode.value) {
      store.addWaypoint(e.latlng.lat, e.latlng.lng);
    }
  });

  map.on('zoomend', () => {
    if (map) {
      currentZoom.value = map.getZoom();
      drawWaypoints();
      drawSegmentLabels();
    }
  });
}

function drawNoFlyZones() {
  if (!zoneLayer) return;
  zoneLayer.clearLayers();
  for (const zone of store.noFlyZones) {
    const color =
      zone.type === 'airport' ? '#ef4444' :
      zone.type === 'military' ? '#f97316' : '#a855f7';
    L.circle([zone.center[0], zone.center[1]], {
      radius: zone.radius,
      color,
      fillColor: color,
      fillOpacity: 0.15,
      weight: 2,
    })
      .bindPopup(`<b>${zone.name}</b><br>Type: ${zone.type}<br>Radius: ${zone.radius}m`)
      .addTo(zoneLayer);
  }
}

function createActionIcon(action: string, idx: number): L.DivIcon {
  const meta = ACTION_META[action] || ACTION_META.none;
  const density = getZoomDensity();
  const size = density === 'high' ? 28 : density === 'medium' ? 22 : 18;
  const fontSize = density === 'high' ? 14 : density === 'medium' ? 11 : 9;
  const showLabel = density !== 'low';
  const seqLabel = showLabel ? `<span style="font-size:${fontSize - 2}px;color:#94a3b8;margin-left:2px">WP${idx + 1}</span>` : '';

  return L.divIcon({
    className: 'action-icon-wrapper',
    html: `
      <div style="
        width:${size}px;height:${size}px;
        background:${meta.color};
        border-radius:50%;
        display:flex;align-items:center;justify-content:center;
        color:#fff;font-size:${fontSize}px;
        box-shadow:0 2px 6px rgba(0,0,0,0.25);
        border:2px solid #fff;
        transform: translate(-50%, -50%);
      ">
        ${meta.icon}
      </div>
      <div style="
        position:absolute;
        top:${size + 2}px;
        left:50%;
        transform:translateX(-50%);
        white-space:nowrap;
        background:rgba(15,23,42,0.85);
        color:#e2e8f0;
        padding:1px 6px;
        border-radius:4px;
        font-size:${fontSize - 1}px;
        border:1px solid rgba(148,163,184,0.2);
        display:${showLabel ? 'block' : 'none'};
      ">
        ${meta.label}${seqLabel}
      </div>
    `,
    iconSize: [size, size + (showLabel ? 22 : 0)],
    iconAnchor: [size / 2, size / 2],
  });
}

function createSpeedLabel(
  speed: number,
  distance: number,
  segIdx: number,
  angle: number
): L.DivIcon {
  const density = getZoomDensity();
  if (density === 'low') {
    return L.divIcon({
      className: 'speed-label-minimal',
      html: ``,
      iconSize: [0, 0],
      iconAnchor: [0, 0],
    });
  }

  const kmh = Math.round(speed * 3.6);
  const distStr = distance < 1000
    ? `${Math.round(distance)}m`
    : `${(distance / 1000).toFixed(1)}km`;

  const isHigh = density === 'high';
  const padding = isHigh ? '6px 12px' : '4px 8px';
  const fontSize = isHigh ? 12 : 10;
  const showDetail = isHigh;

  let displayAngle = -angle;
  while (displayAngle > 90) displayAngle -= 180;
  while (displayAngle < -90) displayAngle += 180;

  const useRotation = isHigh && Math.abs(displayAngle) <= 35;

  return L.divIcon({
    className: 'speed-segment-label',
    html: `
      <div style="
        background:rgba(15,23,42,0.9);
        color:#e2e8f0;
        padding:${padding};
        border-radius:6px;
        border:1px solid rgba(59,130,246,0.4);
        box-shadow:0 2px 8px rgba(0,0,0,0.3);
        white-space:nowrap;
        font-family:ui-monospace, SFMono-Regular, Menlo, monospace;
        ${useRotation ? `transform: rotate(${displayAngle}deg);` : ''}
        transform-origin: center center;
        pointer-events: auto;
        cursor: help;
      ">
        <div style="display:flex;align-items:center;gap:${isHigh ? '8px' : '4px'};font-size:${fontSize}px">
          <span style="color:#60a5fa">⚡</span>
          <span style="font-weight:600;color:#fbbf24">${speed}${isHigh ? ' m/s' : ''}</span>
          ${showDetail ? `<span style="color:#94a3b8">|</span>` : ''}
          ${showDetail ? `<span style="color:#34d399">${kmh}km/h</span>` : ''}
        </div>
        ${showDetail ? `
        <div style="font-size:${fontSize - 2}px;color:#94a3b8;margin-top:2px;display:flex;gap:6px">
          <span>📏 ${distStr}</span>
          <span>│</span>
          <span>S${segIdx + 1}</span>
        </div>` : ''}
      </div>
    `,
    iconSize: [isHigh ? 140 : 80, isHigh ? 48 : 24],
    iconAnchor: [isHigh ? 70 : 40, isHigh ? 24 : 12],
  });
}

function calculateDistance(
  lat1: number,
  lng1: number,
  lat2: number,
  lng2: number
): number {
  const R = 6371000;
  const toRad = (v: number) => (v * Math.PI) / 180;
  const dLat = toRad(lat2 - lat1);
  const dLng = toRad(lng2 - lng1);
  const a =
    Math.sin(dLat / 2) ** 2 +
    Math.cos(toRad(lat1)) * Math.cos(toRad(lat2)) * Math.sin(dLng / 2) ** 2;
  return 2 * R * Math.asin(Math.sqrt(a));
}

function calculateAngle(
  lat1: number,
  lng1: number,
  lat2: number,
  lng2: number
): number {
  const toRad = (v: number) => (v * Math.PI) / 180;
  const toDeg = (v: number) => (v * 180) / Math.PI;
  const dLng = toRad(lng2 - lng1);
  const y = Math.sin(dLng) * Math.cos(toRad(lat2));
  const x =
    Math.cos(toRad(lat1)) * Math.sin(toRad(lat2)) -
    Math.sin(toRad(lat1)) * Math.cos(toRad(lat2)) * Math.cos(dLng);
  return toDeg(Math.atan2(y, x));
}

function drawWaypoints() {
  if (!waypointLayer) return;
  waypointLayer.clearLayers();
  store.waypoints.forEach((wp, idx) => {
    const actionIcon = createActionIcon(wp.action, idx);
    const marker = L.marker([wp.lat, wp.lng], { icon: actionIcon });

    marker.bindPopup(`
      <div style="min-width:180px">
        <div style="display:flex;align-items:center;gap:6px;margin-bottom:6px">
          <span style="font-size:16px">${ACTION_META[wp.action]?.icon || '•'}</span>
          <b style="font-size:14px">航点 ${idx + 1}</b>
          <span style="font-size:11px;color:${ACTION_META[wp.action]?.color || '#6b7280'};background:${ACTION_META[wp.action]?.color || '#6b7280'}22;padding:1px 6px;border-radius:4px">
            ${ACTION_META[wp.action]?.label || '无动作'}
          </span>
        </div>
        <div style="font-size:12px;line-height:1.7">
          <div>📍 海拔: <b style="color:#60a5fa">${wp.altitude}m</b></div>
          <div>⚡ 速度: <b style="color:#fbbf24">${wp.speed} m/s</b> (${Math.round(wp.speed * 3.6)} km/h)</div>
          <div>🎯 动作: <b style="color:${ACTION_META[wp.action]?.color || '#6b7280'}">${ACTION_META[wp.action]?.label || '无动作'}</b></div>
        </div>
        <button onclick="this.closest('.leaflet-popup').remove();window.__removeWaypoint?.('${wp.id}')" 
          style="margin-top:8px;color:#ef4444;background:fee2e2;border:1px solid #fecaca;padding:4px 10px;border-radius:4px;cursor:pointer;font-size:12px;width:100%">
          删除航点
        </button>
      </div>
    `);

    marker.on('dragend', (e: any) => {
      const ll = e.target.getLatLng();
      store.updateWaypoint(wp.id, { lat: ll.lat, lng: ll.lng });
    });

    if (getZoomDensity() !== 'low') {
      (marker as any).dragging?.enable();
    } else {
      (marker as any).dragging?.disable();
    }

    marker.addTo(waypointLayer!);
  });
}

function drawSegmentLabels() {
  if (!segmentLabelLayer || !map) return;
  segmentLabelLayer.clearLayers();

  const density = getZoomDensity();
  if (density === 'low' || store.waypoints.length < 2) return;

  for (let i = 0; i < store.waypoints.length - 1; i++) {
    const wp1 = store.waypoints[i];
    const wp2 = store.waypoints[i + 1];

    const midLat = (wp1.lat + wp2.lat) / 2;
    const midLng = (wp1.lng + wp2.lng) / 2;
    const segSpeed = wp2.speed;
    const distance = calculateDistance(wp1.lat, wp1.lng, wp2.lat, wp2.lng);
    const angle = calculateAngle(wp1.lat, wp1.lng, wp2.lat, wp2.lng);

    const labelIcon = createSpeedLabel(segSpeed, distance, i, angle);
    const labelMarker = L.marker([midLat, midLng], {
      icon: labelIcon,
      interactive: true,
    });

    const minutes = distance > 0 && segSpeed > 0 ? (distance / segSpeed / 60).toFixed(1) : '0';
    labelMarker.bindTooltip(`
      <div style="font-size:12px">
        <div style="font-weight:600;margin-bottom:4px">航段 S${i + 1}</div>
        <div>⚡ 速度: ${segSpeed} m/s (${Math.round(segSpeed * 3.6)} km/h)</div>
        <div>📏 距离: ${distance < 1000 ? Math.round(distance) + 'm' : (distance / 1000).toFixed(2) + 'km'}</div>
        <div>⏱ 预计: ${minutes} 分钟</div>
      </div>
    `, {
      direction: 'top',
      offset: [0, -10],
      className: 'segment-tooltip',
    });

    labelMarker.addTo(segmentLabelLayer);
  }
}

function drawRoute() {
  if (routeLayer && map) {
    map.removeLayer(routeLayer);
    routeLayer = null;
  }
  if (store.waypoints.length < 2 || !map) return;

  const latlngs = store.waypoints.map((w) => [w.lat, w.lng] as [number, number]);

  let hasDanger = false;
  for (const wp of store.waypoints) {
    for (const zone of store.noFlyZones) {
      const d = Math.sqrt(
        (wp.lat - zone.center[0]) ** 2 + (wp.lng - zone.center[1]) ** 2
      ) * 111000;
      if (d < zone.radius * 1.5) hasDanger = true;
    }
  }

  const density = getZoomDensity();
  const baseWeight = density === 'high' ? 5 : density === 'medium' ? 4 : 3;

  routeLayer = L.polyline(latlngs, {
    color: hasDanger ? '#ef4444' : '#22c55e',
    weight: baseWeight,
    opacity: 0.85,
    dashArray: hasDanger ? '8,4' : undefined,
    lineJoin: 'round',
    lineCap: 'round',
  }).addTo(map);

  const arrowColor = hasDanger ? '#fca5a5' : '#86efac';
  const arrowSize = density === 'high' ? 8 : density === 'medium' ? 6 : 4;
  for (let i = 0; i < latlngs.length - 1; i++) {
    const p1 = latlngs[i];
    const p2 = latlngs[i + 1];
    const midLat = (p1[0] + p2[0]) / 2;
    const midLng = (p1[1] + p2[1]) / 2;
    const angle = calculateAngle(p1[0], p1[1], p2[0], p2[1]);

    const arrowIcon = L.divIcon({
      className: 'arrow-indicator',
      html: `
        <div style="
          width:0;height:0;
          border-left:${arrowSize}px solid transparent;
          border-right:${arrowSize}px solid transparent;
          border-bottom:${arrowSize * 1.6}px solid ${arrowColor};
          transform: translate(-50%, -50%) rotate(${angle + 90}deg);
          filter: drop-shadow(0 1px 2px rgba(0,0,0,0.4));
        "></div>
      `,
      iconSize: [arrowSize * 2, arrowSize * 2],
      iconAnchor: [arrowSize, arrowSize],
    });
    L.marker([midLat, midLng], { icon: arrowIcon, interactive: false })
      .addTo(map);
  }
}

function drawSimDrone() {
  if (!map || store.waypoints.length < 2) return;
  const progress = store.simProgress / 100;
  const totalWp = store.waypoints.length;
  const segIdx = Math.min(Math.floor(progress * (totalWp - 1)), totalWp - 2);
  const segProgress = (progress * (totalWp - 1)) - segIdx;
  const wp1 = store.waypoints[segIdx];
  const wp2 = store.waypoints[segIdx + 1];
  const lat = wp1.lat + (wp2.lat - wp1.lat) * segProgress;
  const lng = wp1.lng + (wp2.lng - wp1.lng) * segProgress;

  if (droneMarker) {
    droneMarker.setLatLng([lat, lng]);
  } else {
    droneMarker = L.circleMarker([lat, lng], {
      radius: 10,
      color: '#fbbf24',
      fillColor: '#f59e0b',
      fillOpacity: 1,
      weight: 3,
    }).addTo(map);
  }
}

(window as any).__removeWaypoint = (id: string) => {
  store.removeWaypoint(id);
};

watch(() => store.waypoints.length, () => {
  drawWaypoints();
  drawRoute();
  drawSegmentLabels();
});

watch(() => store.waypoints, () => {
  drawWaypoints();
  drawRoute();
  drawSegmentLabels();
}, { deep: true });

watch(() => store.noFlyZones.length, drawNoFlyZones);
watch(() => store.simProgress, drawSimDrone);

onMounted(() => {
  nextTick(initMap);
});

onUnmounted(() => {
  delete (window as any).__removeWaypoint;
  if (map) {
    map.remove();
    map = null;
  }
});

function toggleAddMode() {
  addMode.value = !addMode.value;
}

function handlePlanRoute() {
  if (store.waypoints.length < 2) return;
  const first = store.waypoints[0];
  const last = store.waypoints[store.waypoints.length - 1];
  store.planRoute([first.lat, first.lng], [last.lat, last.lng]);
}
</script>

<template>
  <div class="relative w-full h-full">
    <div ref="mapContainer" class="w-full h-full rounded-lg" />
    <div class="absolute top-2 right-2 z-[1000] flex flex-col gap-1">
      <button
        @click="toggleAddMode"
        :class="addMode ? 'bg-blue-600 text-white' : 'bg-gray-800 text-gray-300'"
        class="px-3 py-1 rounded text-xs font-medium shadow hover:opacity-90 transition"
      >
        {{ addMode ? '✦ 添加模式' : '○ 点击添加' }}
      </button>
      <button
        @click="handlePlanRoute"
        class="px-3 py-1 rounded text-xs font-medium bg-green-700 text-white shadow hover:opacity-90 transition"
      >
        规划航线
      </button>
      <button
        @click="store.clearRoute()"
        class="px-3 py-1 rounded text-xs font-medium bg-red-700 text-white shadow hover:opacity-90 transition"
      >
        清除
      </button>
    </div>
    <div class="absolute bottom-2 left-2 z-[1000] bg-slate-900/85 backdrop-blur-sm text-gray-200 text-xs px-3 py-2 rounded-lg border border-slate-700 shadow-lg">
      <div class="font-semibold mb-1 text-slate-100">📋 图例</div>
      <div class="flex flex-wrap gap-x-3 gap-y-1 text-[11px]">
        <span class="flex items-center gap-1"><span style="color:#f59e0b">⏸</span> 悬停</span>
        <span class="flex items-center gap-1"><span style="color:#3b82f6">📷</span> 拍照</span>
        <span class="flex items-center gap-1"><span style="color:#8b5cf6">🎬</span> 录像</span>
        <span class="flex items-center gap-1"><span style="color:#6b7280">•</span> 无动作</span>
      </div>
      <div class="mt-1.5 pt-1.5 border-t border-slate-700/50 text-[10px] text-slate-400">
        缩放级别: <span class="text-blue-400 font-medium">{{ currentZoom }}</span>
        <span class="mx-1.5">|</span>
        显示模式: <span class="text-emerald-400 font-medium">
          {{ currentZoom >= 15 ? '详细' : currentZoom >= 12 ? '标准' : '概览' }}
        </span>
      </div>
    </div>
  </div>
</template>

<style scoped>
:deep(.wp-tooltip) {
  background: rgba(30, 41, 59, 0.9);
  color: #e2e8f0;
  border: 1px solid #475569;
  font-size: 10px;
  padding: 1px 4px;
  border-radius: 4px;
}

:deep(.action-icon-wrapper) {
  background: transparent !important;
  border: none !important;
}

:deep(.speed-segment-label) {
  background: transparent !important;
  border: none !important;
}

:deep(.speed-segment-label:hover > div) {
  background: rgba(30, 58, 138, 0.95) !important;
  border-color: rgba(96, 165, 250, 0.8) !important;
  box-shadow: 0 4px 16px rgba(59, 130, 246, 0.4) !important;
}

:deep(.arrow-indicator) {
  background: transparent !important;
  border: none !important;
}

:deep(.segment-tooltip) {
  background: rgba(15, 23, 42, 0.95);
  color: #e2e8f0;
  border: 1px solid rgba(59, 130, 246, 0.5);
  border-radius: 6px;
  padding: 4px 8px;
  font-size: 11px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.3);
}

:deep(.segment-tooltip::before) {
  border-top-color: rgba(59, 130, 246, 0.5);
}
</style>
