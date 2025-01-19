<template>
  <div class="container">
    <h2>Cell Distribution (CD45+)</h2>
    <div class="toolbar">
      <button @click="startDrawing" :disabled="isDrawing">Arbitrary Polygon</button>
      <button @click="finishPolygon" :disabled="!isDrawing">Finish Polygon</button>
      <button @click="toggleAllPolygonsVisibility">Show / Hide All Polygons</button>
    </div>
    <div class="chart-container" :style="{ width: outerWidth + 'px', height: outerHeight + 'px' }">
      <canvas
        ref="canvasRef"
        :width="outerWidth"
        :height="outerHeight"
        class="overlay-layer"
      ></canvas>
      <svg
        :width="outerWidth"
        :height="outerHeight"
        class="overlay-layer"
        @click="handleSvgClick"
      >
        <g :transform="`translate(${margin.left}, ${margin.top})`">
          <g v-for="(poly, idx) in polygons" :key="idx">
            <polygon
              v-if="poly.visible"
              :points="formatPoints(poly.points)"
              :fill="poly.fillColor"
              :stroke="poly.strokeColor"
              :stroke-dasharray="poly.strokeDasharray"
              fill-opacity="0.2"
              @click.stop="openEditPolygon(idx)"
            />
            <text
              v-if="poly.visible"
              :x="polygonLabelX(poly.points)"
              :y="polygonLabelY(poly.points)"
              fill="black"
              font-weight="bold"
            >
              {{ poly.label }}
              ({{ poly.stats.count }}/{{ totalPoints }} - {{ poly.stats.percent }}%)
            </text>
          </g>
          <polygon
            v-if="isDrawing && drawingPoints.length > 1"
            :points="formatPoints(drawingPoints)"
            fill="rgba(255,0,0,0.2)"
            stroke="red"
          />
          <g ref="xAxisRef" :transform="`translate(0, ${innerHeight})`"></g>
          <g ref="yAxisRef"></g>
          <text
            :x="innerWidth / 2"
            :y="innerHeight + 40"
            text-anchor="middle"
            font-weight="bold"
          >
            CD45-KrO
          </text>
          <text
            :transform="`rotate(-90)`"
            :x="-(innerHeight / 2)"
            :y="-45"
            text-anchor="middle"
            font-weight="bold"
          >
            SS INT LIN
          </text>
        </g>
      </svg>
    </div>
    <div
      v-if="editIndex !== null"
      class="modal-overlay"
      @click.self="cancelPolygonEdits"
    >
      <div class="modal-content">
        <h3>Edit Polygon</h3>
        <label>
          Label:
          <select v-model="editForm.selectedPreset">
            <option value="">(無)</option>
            <option value="CD45-">CD45-</option>
            <option value="Gr">Gr</option>
            <option value="Mo">Mo</option>
            <option value="Ly">Ly</option>
          </select>
        </label>
        <label>
          Name:
          <input type="text" v-model="editForm.customName" />
        </label>
        <label>
          Fill Color:
          <input type="color" v-model="editForm.fillColor" />
        </label>
        <label>
          Stroke Color:
          <input type="color" v-model="editForm.strokeColor" />
        </label>
        <label>
          Line Style:
          <select v-model="editForm.strokeDasharray">
            <option value="">Solid</option>
            <option value="5,5">Dashed (5,5)</option>
            <option value="2,2">Dotted (2,2)</option>
          </select>
        </label>
        <div>
          <button @click="savePolygonEdits">Save</button>
          <button @click="cancelPolygonEdits">Cancel</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';
import * as d3 from 'd3';
import { polygonContains } from 'd3-polygon';

const outerWidth = 900;
const outerHeight = 700;
const margin = { top: 20, right: 20, bottom: 60, left: 60 };
const innerWidth = outerWidth - margin.left - margin.right;
const innerHeight = outerHeight - margin.top - margin.bottom;

const isDrawing = ref(false);
const drawingPoints = ref([]);

const dataPoints = ref([]);
const polygons = ref([]);

const editIndex = ref(null);
const editForm = ref({
  selectedPreset: '',
  customName: '',
  fillColor: '#ff0000',
  strokeColor: '#ff0000',
  strokeDasharray: ''
});

const PRESETS = ['CD45-', 'Gr', 'Mo', 'Ly'];

let xScale, yScale;
const xAxisRef = ref(null);
const yAxisRef = ref(null);
const canvasRef = ref(null);
const totalPoints = ref(0);

onMounted(async () => {
  const rawData = await d3.csv('/data/CD45_pos.csv');
  const parsed = rawData.map(d => ({
    xVal: +d['CD45-KrO'],
    yVal: +d['SS INT LIN']
  }));
  totalPoints.value = parsed.length;

  xScale = d3.scaleLinear().domain([200, 1000]).range([0, innerWidth]);
  yScale = d3.scaleLinear().domain([0, 1000]).range([innerHeight, 0]);

  dataPoints.value = parsed.map(d => ({
    x: xScale(d.xVal),
    y: yScale(d.yVal)
  }));

  drawCanvasPoints();
  drawAxes();
})

const drawCanvasPoints = () => {
  const ctx = canvasRef.value.getContext('2d');
  ctx.clearRect(0,0,outerWidth,outerHeight);
  ctx.fillStyle = 'gray';
  dataPoints.value.forEach(pt => {
    const cx = pt.x + margin.left;
    const cy = pt.y + margin.top;
    ctx.beginPath();
    ctx.arc(cx, cy, 2, 0, 2*Math.PI);
    ctx.fill();
  });
};

const drawAxes = () => {
  const xAxis = d3.axisBottom(xScale).ticks(8);
  const yAxis = d3.axisLeft(yScale).ticks(10);
  d3.select(xAxisRef.value).call(xAxis);
  d3.select(yAxisRef.value).call(yAxis);
};

const startDrawing = () => {
  isDrawing.value = true;
  drawingPoints.value = [];
};

const handleSvgClick = (e) => {
  if (!isDrawing.value) return;
  const rect = e.currentTarget.getBoundingClientRect();
  const mouseX = e.clientX - rect.left - margin.left;
  const mouseY = e.clientY - rect.top - margin.top;
  drawingPoints.value.push([mouseX, mouseY]);
};

const finishPolygon = () => {
  if (!isDrawing.value) return;
  isDrawing.value = false;

  if (drawingPoints.value.length < 3) {
    drawingPoints.value = [];
    return;
  }

  const newPoly = {
    points: [...drawingPoints.value],
    label: 'New Polygon',
    fillColor: '#ff0000',
    strokeColor: '#ff0000',
    strokeDasharray: '',
    visible: true,
    stats: { count: 0, percent: 0 }
  };
  updatePolygonStats(newPoly);
  polygons.value.push(newPoly);
  drawingPoints.value = [];
};

const updatePolygonStats = (poly) => {
  let xs = poly.points.map(p => p[0]);
  let ys = poly.points.map(p => p[1]);
  const minX = Math.min(...xs), maxX = Math.max(...xs);
  const minY = Math.min(...ys), maxY = Math.max(...ys);

  let count = 0;
  dataPoints.value.forEach(pt => {
    if (pt.x < minX || pt.x > maxX || pt.y < minY || pt.y > maxY) {
      return;
    }
    if (polygonContains(poly.points, [pt.x, pt.y])) {
      count++;
    }
  });
  const total = totalPoints.value || 1;
  poly.stats.count = count;
  poly.stats.percent = ((count / total) * 100).toFixed(1);
};


const toggleAllPolygonsVisibility = () => {
  polygons.value.forEach(p => {
    p.visible = !p.visible;
  });
};

const openEditPolygon = (idx) => {
  editIndex.value = idx;
  const poly = polygons.value[idx];

  editForm.value.fillColor = poly.fillColor;
  editForm.value.strokeColor = poly.strokeColor;
  editForm.value.strokeDasharray = poly.strokeDasharray;

  let foundPreset = '';
  let foundName = poly.label;

  for (const p of PRESETS) {
    const prefix = p + ' ';
    if (poly.label.startsWith(prefix)) {
      foundPreset = p;
      foundName = poly.label.slice(prefix.length);
      break;
    }
  }

  editForm.value.selectedPreset = foundPreset;
  editForm.value.customName = foundName;
};

const savePolygonEdits = () => {
  if (editIndex.value === null) return;
  const idx = editIndex.value;
  const poly = polygons.value[idx];

  let finalLabel = editForm.value.customName;
  if (editForm.value.selectedPreset) {
    finalLabel = editForm.value.selectedPreset + (finalLabel ? ' ' + finalLabel : '');
  }

  poly.label = finalLabel || 'New Polygon';
  poly.fillColor = editForm.value.fillColor;
  poly.strokeColor = editForm.value.strokeColor;
  poly.strokeDasharray = editForm.value.strokeDasharray;

  editIndex.value = null;
};

const cancelPolygonEdits = () => {
  editIndex.value = null;
};

const formatPoints = (pts) => {
  return pts.map(p => p.join(',')).join(' ');
};

const polygonLabelX = (pts) => {
  const xs = pts.map(p => p[0]);
  return d3.mean(xs) || 0;
};

const polygonLabelY = (pts) => {
  const ys = pts.map(p => p[1]);
  return d3.mean(ys) || 0;
};
</script>

<style scoped>
.container {
  width: 900px;
  margin: auto;
}
.toolbar {
  margin-bottom: 10px;
}
.chart-container {
  position: relative;
  border: 1px solid #ccc;
}
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 10;
}
.modal-content {
  background: #fff;
  padding: 20px;
  border-radius: 4px;
  min-width: 300px;
  box-shadow: 0 2px 10px rgba(0,0,0,0.3);
}
.modal-content label {
  display: block;
  margin: 5px 0;
}
button {
  margin-top: 10px;
  margin-right: 10px;
}
h3 {
  margin-top: 0px;
}
.overlay-layer {
  position: absolute;
  top: 0;
  left: 0;
  z-index: 1; 
}
</style>
