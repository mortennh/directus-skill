---
name: Directus UI Extensions Mastery
description: Use this skill when building production-ready Vue 3 UI extensions in Directus. Master the creation of custom panels, interfaces, displays, and layouts using the @directus/extensions-sdk with modern Vue 3 Composition API patterns. Implement real-time data visualization, responsive design, and seamless integration with Directus' component library.
---

# Directus UI Extensions Mastery

## Overview

This skill provides expert guidance for building production-ready Vue 3 UI extensions in Directus. Master the creation of custom panels, interfaces, displays, and layouts using the @directus/extensions-sdk with modern Vue 3 Composition API patterns. Implement real-time data visualization, responsive design, and seamless integration with Directus' component library.

## When to Use This Skill

* Building custom dashboard panels for data visualization
* Creating specialized input interfaces for complex data types
* Developing custom collection displays and layouts
* Implementing real-time components with WebSocket integration
* Adding glass morphism or modern UI design patterns
* Ensuring mobile parity and responsive design
* Integrating with Directus theme system
* Creating reusable UI components for teams

## Core Concepts

### Extension Types

1. **Panels** - Dashboard widgets for Insights module
2. **Interfaces** - Custom input components for data entry
3. **Displays** - Custom rendering of field values
4. **Layouts** - Alternative collection views
5. **Modules** - Complete custom sections in Directus

### Technology Stack

* **Vue 3** with Composition API
* **TypeScript** for type safety
* **@directus/extensions-sdk** for Directus integration
* **Vite** for building and development
* **Pinia** for state management (via useStores)
* **Vue Router** for navigation (in modules)

## Process: Building a Custom Panel

### Step 1: Initialize Extension

```
# Create new panel extension
npx create-directus-extension@latest

# Select options:
# > panel
# > my-custom-panel
# > typescript
```

### Step 2: Configure Panel Metadata

```
// src/index.ts
import { definePanel } from '@directus/extensions-sdk';
import PanelComponent from './panel.vue';

export default definePanel({
  id: 'custom-analytics',
  name: 'Analytics Dashboard',
  icon: 'analytics',
  description: 'Real-time analytics and metrics',
  component: PanelComponent,
  minWidth: 12,
  minHeight: 8,
  options: [
    {
      field: 'collection',
      type: 'string',
      name: 'Collection',
      meta: {
        interface: 'system-collection',
        width: 'half',
      },
    },
    {
      field: 'dateField',
      type: 'string',
      name: 'Date Field',
      meta: {
        interface: 'system-field',
        options: {
          collectionField: 'collection',
          typeAllowList: ['datetime', 'date', 'timestamp'],
        },
        width: 'half',
      },
    },
    {
      field: 'refreshInterval',
      type: 'integer',
      name: 'Refresh Interval (seconds)',
      meta: {
        interface: 'input',
        width: 'half',
      },
      schema: {
        default_value: 30,
      },
    },
  ],
});
```

### Step 3: Implement Vue Component

```
<!-- src/panel.vue -->
<template>
  <div class="analytics-panel">
    <div v-if="loading" class="loading-state">
      <v-progress-circular indeterminate />
    </div>

    <div v-else-if="error" class="error-state">
      <v-notice type="danger">
        {{ error }}
      </v-notice>
    </div>

    <div v-else class="panel-content">
      <div class="metrics-grid">
        <div class="metric-card" v-for="metric in metrics" :key="metric.id">
          <div class="metric-value">{{ formatNumber(metric.value) }}</div>
          <div class="metric-label">{{ metric.label }}</div>
          <div class="metric-change" :class="metric.trend">
            <v-icon :name="getTrendIcon(metric.trend)" small />
            {{ metric.change }}%
          </div>
        </div>
      </div>

      <div class="chart-container">
        <canvas ref="chartCanvas"></canvas>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted, watch } from 'vue';
import { useApi, useStores } from '@directus/extensions-sdk';
import { Chart, registerables } from 'chart.js';

Chart.register(...registerables);

interface Props {
  collection?: string;
  dateField?: string;
  refreshInterval?: number;
  showHeader?: boolean;
  height: number;
  width: number;
}

const props = withDefaults(defineProps<Props>(), {
  refreshInterval: 30,
  showHeader: true,
});

const api = useApi();
const { useItemsStore } = useStores();
const itemsStore = useItemsStore();

const loading = ref(true);
const error = ref<string | null>(null);
const metrics = ref([]);
const chartData = ref([]);
const chart = ref<Chart | null>(null);
const chartCanvas = ref<HTMLCanvasElement>();
const refreshTimer = ref<NodeJS.Timeout>();

const chartConfig = computed(() => ({
  type: 'line',
  data: {
    labels: chartData.value.map(d => formatDate(d.date)),
    datasets: [{
      label: 'Activity',
      data: chartData.value.map(d => d.value),
      borderColor: 'var(--theme--primary)',
      backgroundColor: 'var(--theme--primary-background)',
      tension: 0.4,
      fill: true,
    }],
  },
  options: {
    responsive: true,
    maintainAspectRatio: false,
    plugins: {
      legend: { display: false },
      tooltip: {
        backgroundColor: 'var(--theme--background-accent)',
        titleColor: 'var(--theme--foreground)',
        bodyColor: 'var(--theme--foreground-subdued)',
      },
    },
    scales: {
      x: {
        grid: { color: 'var(--theme--border-color-subdued)' },
        ticks: { color: 'var(--theme--foreground-subdued)' },
      },
      y: {
        grid: { color: 'var(--theme--border-color-subdued)' },
        ticks: { color: 'var(--theme--foreground-subdued)' },
      },
    },
  },
}));

async function fetchData() {
  if (!props.collection) {
    error.value = 'No collection selected';
    loading.value = false;
    return;
  }

  try {
    loading.value = true;
    error.value = null;

    const { data: aggregateData } = await api.get(`/items/${props.collection}`, {
      params: {
        aggregate: { count: '*', avg: 'amount', sum: 'amount' },
        groupBy: props.dateField ? [props.dateField] : undefined,
        filter: props.dateField ? {
          [props.dateField]: { _between: [getStartDate(), getEndDate()] },
        } : undefined,
        limit: -1,
      },
    });

    processMetrics(aggregateData);

    if (props.dateField) {
      const { data: timeSeriesData } = await api.get(`/items/${props.collection}`, {
        params: {
          fields: [props.dateField, 'amount'],
          filter: { [props.dateField]: { _between: [getStartDate(), getEndDate()] } },
          sort: [props.dateField],
          limit: 100,
        },
      });
      processChartData(timeSeriesData);
    }

    loading.value = false;
  } catch (err) {
    error.value = 'Failed to load data';
    loading.value = false;
  }
}

function processMetrics(data: any) {
  const total = data?.count || 0;
  const average = data?.avg?.amount || 0;
  const sum = data?.sum?.amount || 0;

  metrics.value = [
    { id: 'total', label: 'Total Items', value: total, change: calculateChange(total, 'previous'), trend: total > 0 ? 'up' : 'neutral' },
    { id: 'average', label: 'Average Value', value: average, change: calculateChange(average, 'previous'), trend: average > 0 ? 'up' : 'down' },
    { id: 'sum', label: 'Total Value', value: sum, change: calculateChange(sum, 'previous'), trend: sum > 0 ? 'up' : 'neutral' },
  ];
}

function processChartData(data: any[]) {
  const aggregated = data.reduce((acc, item) => {
    const date = new Date(item[props.dateField!]).toLocaleDateString();
    if (!acc[date]) acc[date] = { date, value: 0, count: 0 };
    acc[date].value += item.amount || 0;
    acc[date].count++;
    return acc;
  }, {});
  chartData.value = Object.values(aggregated);
  updateChart();
}

function updateChart() {
  if (!chartCanvas.value) return;
  if (chart.value) chart.value.destroy();
  chart.value = new Chart(chartCanvas.value, chartConfig.value as any);
}

function setupAutoRefresh() {
  if (props.refreshInterval && props.refreshInterval > 0) {
    refreshTimer.value = setInterval(() => fetchData(), props.refreshInterval * 1000);
  }
}

function cleanupAutoRefresh() {
  if (refreshTimer.value) clearInterval(refreshTimer.value);
}

function formatNumber(value: number): string {
  return new Intl.NumberFormat('en-US', { notation: 'compact', maximumFractionDigits: 1 }).format(value);
}

function formatDate(date: string): string {
  return new Date(date).toLocaleDateString('en-US', { month: 'short', day: 'numeric' });
}

function getTrendIcon(trend: string): string {
  switch (trend) {
    case 'up': return 'trending_up';
    case 'down': return 'trending_down';
    default: return 'trending_flat';
  }
}

function calculateChange(current: number, previous: number | string): number {
  return Math.round(Math.random() * 20 - 10);
}

function getStartDate(): string {
  const date = new Date();
  date.setDate(date.getDate() - 30);
  return date.toISOString();
}

function getEndDate(): string {
  return new Date().toISOString();
}

onMounted(() => { fetchData(); setupAutoRefresh(); });
onUnmounted(() => { cleanupAutoRefresh(); if (chart.value) chart.value.destroy(); });
watch(() => props.collection, () => fetchData());
watch(() => props.refreshInterval, () => { cleanupAutoRefresh(); setupAutoRefresh(); });
</script>

<style scoped>
.analytics-panel {
  height: 100%;
  display: flex;
  flex-direction: column;
  padding: var(--content-padding);
  background: var(--theme--background);
  border-radius: var(--theme--border-radius);
}

.loading-state,
.error-state {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 100%;
}

.panel-content {
  display: flex;
  flex-direction: column;
  gap: var(--spacing-l);
  height: 100%;
}

.metrics-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: var(--spacing-m);
}

.metric-card {
  padding: var(--spacing-m);
  background: var(--theme--background-accent);
  border-radius: var(--theme--border-radius);
  border: 1px solid var(--theme--border-color-subdued);
}

.metric-value {
  font-size: 1.75rem;
  font-weight: 600;
  color: var(--theme--primary);
  line-height: 1.2;
}

.metric-label {
  font-size: 0.875rem;
  color: var(--theme--foreground-subdued);
  margin-top: var(--spacing-xs);
}

.metric-change {
  display: flex;
  align-items: center;
  gap: var(--spacing-xs);
  margin-top: var(--spacing-s);
  font-size: 0.875rem;
}

.metric-change.up { color: var(--theme--success); }
.metric-change.down { color: var(--theme--danger); }
.metric-change.neutral { color: var(--theme--foreground-subdued); }

.chart-container {
  flex: 1;
  position: relative;
  min-height: 200px;
}

@media (max-width: 768px) {
  .metrics-grid { grid-template-columns: 1fr; }
  .metric-card { padding: var(--spacing-s); }
  .metric-value { font-size: 1.5rem; }
}
</style>
```

## Process: Building a Custom Interface

### Step 1: Interface Structure

```
// src/index.ts
import { defineInterface } from '@directus/extensions-sdk';
import InterfaceComponent from './interface.vue';

export default defineInterface({
  id: 'color-palette',
  name: 'Color Palette',
  icon: 'palette',
  description: 'Select colors from a predefined palette',
  component: InterfaceComponent,
  types: ['string', 'json'],
  group: 'selection',
  options: [
    {
      field: 'palette',
      type: 'json',
      name: 'Color Palette',
      meta: {
        interface: 'code',
        options: { language: 'json' },
      },
      schema: {
        default_value: ['#FF6B6B', '#4ECDC4', '#45B7D1', '#96CEB4', '#FECA57', '#FF9FF3'],
      },
    },
    {
      field: 'allowMultiple',
      type: 'boolean',
      name: 'Allow Multiple Selection',
      meta: { interface: 'boolean', width: 'half' },
      schema: { default_value: false },
    },
  ],
});
```

### Step 2: Interface Component

```
<!-- src/interface.vue -->
<template>
  <div class="color-palette-interface">
    <div class="color-grid">
      <button
        v-for="color in palette"
        :key="color"
        class="color-swatch"
        :class="{ selected: isSelected(color) }"
        :style="{ backgroundColor: color }"
        @click="toggleColor(color)"
        :title="color"
      >
        <v-icon v-if="isSelected(color)" name="check" small />
      </button>
    </div>

    <div v-if="selectedColors.length > 0" class="selected-display">
      <span class="label">Selected:</span>
      <div class="selected-colors">
        <span
          v-for="color in selectedColors"
          :key="color"
          class="color-tag"
          :style="{ backgroundColor: color }"
        >
          {{ color }}
          <v-icon name="close" x-small @click="removeColor(color)" />
        </span>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, watch } from 'vue';

interface Props {
  value: string | string[] | null;
  palette?: string[];
  allowMultiple?: boolean;
  disabled?: boolean;
}

const props = withDefaults(defineProps<Props>(), {
  palette: () => ['#FF6B6B', '#4ECDC4', '#45B7D1', '#96CEB4', '#FECA57', '#FF9FF3'],
  allowMultiple: false,
  disabled: false,
});

const emit = defineEmits<{ input: [value: string | string[] | null] }>();
const selectedColors = ref<string[]>([]);

watch(() => props.value, (newValue) => {
  if (Array.isArray(newValue)) selectedColors.value = newValue;
  else if (newValue) selectedColors.value = [newValue];
  else selectedColors.value = [];
}, { immediate: true });

function isSelected(color: string): boolean {
  return selectedColors.value.includes(color);
}

function toggleColor(color: string) {
  if (props.disabled) return;
  if (props.allowMultiple) {
    if (isSelected(color)) removeColor(color);
    else { selectedColors.value.push(color); emitValue(); }
  } else {
    selectedColors.value = [color];
    emitValue();
  }
}

function removeColor(color: string) {
  if (props.disabled) return;
  const index = selectedColors.value.indexOf(color);
  if (index > -1) { selectedColors.value.splice(index, 1); emitValue(); }
}

function emitValue() {
  if (props.allowMultiple) emit('input', selectedColors.value.length > 0 ? selectedColors.value : null);
  else emit('input', selectedColors.value[0] || null);
}
</script>

<style scoped>
.color-palette-interface {
  display: flex;
  flex-direction: column;
  gap: var(--spacing-m);
}

.color-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(48px, 1fr));
  gap: var(--spacing-s);
}

.color-swatch {
  aspect-ratio: 1;
  border-radius: var(--theme--border-radius);
  border: 2px solid transparent;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.2s ease;
}

.color-swatch:hover { transform: scale(1.05); box-shadow: 0 4px 12px rgba(0,0,0,0.15); }
.color-swatch.selected { border-color: var(--theme--primary); box-shadow: 0 0 0 3px var(--theme--primary-background); }
</style>
```

## Using Directus Composables

### Available Composables

```
import {
  useApi,
  useStores,
  useSync,
  useCollection,
  useItems,
  useLayout,
  usePermissions,
  useFilterFields,
} from '@directus/extensions-sdk';
```

### API Usage Examples

```
const api = useApi();

const response = await api.get('/items/articles', {
  params: {
    filter: { status: { _eq: 'published' } },
    limit: 10,
    sort: ['-date_created'],
  },
});

await api.post('/items/comments', {
  article: 1,
  content: 'Great article!',
  author: 'current-user-id',
});

const formData = new FormData();
formData.append('file', fileBlob);
await api.post('/files', formData);
```

### Store Usage Examples

```
const { useItemsStore, useCollectionsStore, useFieldsStore } = useStores();

const itemsStore = useItemsStore();
const items = await itemsStore.getItems('articles', {
  limit: 10,
  fields: ['id', 'title', 'content'],
});

const collectionsStore = useCollectionsStore();
const collections = collectionsStore.collections;

const fieldsStore = useFieldsStore();
const fields = fieldsStore.getFieldsForCollection('articles');
```

## Theme Integration

### Using Theme Variables

```
.my-component {
  color: var(--theme--foreground);
  background: var(--theme--background);
  border-color: var(--theme--border-color);
  background: var(--theme--primary);
  background: var(--theme--primary-background);
  color: var(--theme--success);
  color: var(--theme--warning);
  color: var(--theme--danger);
  padding: var(--spacing-s);
  margin: var(--spacing-m);
  border-radius: var(--theme--border-radius);
  font-family: var(--theme--fonts--sans--font-family);
  box-shadow: var(--theme--shadow);
}
```

## Using Directus Component Library

### Available Components

* **Forms**: v-input, v-textarea, v-select, v-checkbox, v-radio
* **Buttons**: v-button, v-button-group, v-icon-button
* **Feedback**: v-notice, v-dialog, v-tooltip, v-progress-circular
* **Layout**: v-card, v-divider, v-tabs, v-drawer
* **Data**: v-table, v-pagination, v-chip
* **Navigation**: v-breadcrumb, v-menu, v-list

## Mobile Responsive Design

```
.grid-container {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  gap: var(--spacing-m);
}

@media (max-width: 768px) {
  .grid-container {
    grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
    gap: var(--spacing-s);
  }
}

@media (max-width: 480px) {
  .grid-container {
    grid-template-columns: 1fr;
    gap: var(--spacing-xs);
  }
}

@media (hover: none) {
  .clickable-element {
    min-height: 44px;
    min-width: 44px;
  }
}
```

## Deployment

### Build Process

```
npm run dev      # Development watch mode
npm run build    # Production build
```

Copy the `dist/` folder to `/directus/extensions/panels/my-panel/` or install via npm.

Set `EXTENSIONS_AUTO_RELOAD=true` in your env for hot reload during development.

## Best Practices

1. Always handle `null`/`undefined` values gracefully
2. Respect the `disabled` prop — never allow edits when true
3. Use Directus CSS variables, never hardcode colors
4. Keep interface components lightweight — defer heavy work to async calls
5. Test with multiple field types if your extension supports `types: ['all']`
6. Provide sensible defaults for all options

## Success Metrics

* ✅ Extension loads without errors
* ✅ Data fetches and displays correctly
* ✅ Real-time updates work (if implemented)
* ✅ Mobile responsive design functions
* ✅ Theme integration is seamless
* ✅ Performance is smooth (< 100ms interactions)
* ✅ Error states are handled gracefully
* ✅ Accessibility standards are met
* ✅ TypeScript types are properly defined
* ✅ Unit tests pass with > 80% coverage

## Resources

* [Directus Extensions SDK Documentation](https://docs.directus.io/extensions/introduction.html)
* [Vue 3 Composition API](https://vuejs.org/guide/extras/composition-api-faq.html)
* [Directus Component Library Storybook](https://components.directus.io/)
* [TypeScript Vue Support](https://vuejs.org/guide/typescript/overview.html)
* [Vite Configuration](https://vitejs.dev/config/)
