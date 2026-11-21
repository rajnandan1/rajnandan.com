+++
title = "Largest Triangle Three Buckets: Downsampling Time-Series Data Without Losing Signal"
date = 2026-01-23
description = "How LTTB preserves visual fidelity when rendering millions of data points, with practical TypeScript implementation and real-world trade-offs."
[taxonomies]
tags = ["typescript", "algorithms", "data-visualization", "performance"]
[extra]
toc = true
+++

## The Problem

You have 10 million time-series data points: stock prices, server metrics, IoT sensor readings. Your charting library chokes past 50,000 points, the tab freezes, and users complain.

The naive fix, taking every Nth point, produces jagged lines and drops the peaks and valleys where anomalies live.

Largest Triangle Three Buckets (LTTB) splits your data into buckets and keeps, from each bucket, the one point that forms the largest triangle with its neighbors. That keeps the visual shape of your data with far fewer points, so the chart renders fast.

There is a [live demo](/demos/lttb/) of the algorithm: drag the threshold slider (the number of points to keep), switch between LTTB and every-Nth sampling, and watch which points survive.

## What is LTTB?

Sveinn Steinarsson introduced LTTB in his 2013 master's thesis at the University of Iceland, "Downsampling Time Series for Visual Representation". It is a downsampling algorithm that reduces N points to M points (where M << N) while maintaining visual accuracy. The name describes how it works:

- **Largest Triangle:** For each bucket, it keeps the point that forms the largest triangle with two neighbors: the point it kept from the previous bucket and the average of the next bucket
- **Three Buckets:** It splits the points into buckets, which are runs of consecutive points of about the same count. Each triangle draws on three of them: the previous bucket (its kept point), the current bucket (the candidates) and the next bucket (its average)

Imagine drawing a mountain range: you pick the peaks, valleys, and slopes that capture its shape instead of tracing every grain of sand. LTTB does this mathematically.

### The Triangle Area

Each triangle has three corners: A, the point kept from the previous bucket (for the first bucket, A is the first data point); B, a candidate point in the current bucket; and C, the average of the next bucket. C is an average because the next bucket's point is not chosen yet.

The area is half the length of the base A–C times the height, and the height is how far B sits from the straight line between A and C. So a large area means B is far off that line, which marks a significant visual change, such as a sharp turn, a peak, a valley or a change in trend. Those changes are what your eye needs to read the data's shape.

For the three corners A = (x₁, y₁), B = (x₂, y₂) and C = (x₃, y₃), the area formula is:

```
area = |x₁(y₂ - y₃) + x₂(y₃ - y₁) + x₃(y₁ - y₂)| / 2
```

LTTB computes this area for every candidate B in the bucket and keeps the one with the largest area.

## Why Use LTTB?

### 1. Visual Fidelity Over Statistical Precision

LTTB optimizes for what you see rather than for statistical accuracy, because a rendered chart needs the shape of the data and not each individual point. In the downsampled output:

- Peaks and valleys usually stay visible. ("When You Need Exact Points" shows a case where a peak is dropped.)
- Changes in slope stay clear.

### 2. Predictable Performance

The algorithm expects input that is already sorted by x. It makes one pass through that data and does no sorting of its own, so it runs in O(n) time. It builds no extra data structures, so it needs O(1) space beyond the output buffer. Downsampling 1 million points to 1,000 takes about 3.4ms on an Apple M4 (see the benchmark under "Processing Cost").

### 3. Deterministic Results

Unlike random sampling, LTTB always produces the same output for the same input. Charts look identical across page reloads, a bug in a chart shows up the same way each time you reproduce it, and you can cache downsampled results reliably.

## The LTTB Algorithm: Step by Step

**Setup:**
1. You have N source points
2. You want M target points (M < N). The code calls M the `threshold`.
3. First and last points are always included, so the chart keeps the full x range of the data

**Process:**
1. Divide remaining N-2 points into M-2 buckets
2. Always include the first point
3. For each bucket:
   - Look at the previously selected point (from bucket i-1)
   - Look at the average point of the next bucket (bucket i+1)
   - Select the point in current bucket (bucket i) that forms the largest triangle with these two points
4. Always include the last point

**Visual representation:**

```
First     Bucket 1   Bucket 2   Bucket 3   Bucket 4   Last
point      [...]      [...]      [...]      [...]     point
  ↓          ↓          ↓          ↓          ↓         ↓
kept       Pick 1     Pick 2     Pick 3     Pick 4    kept

For Bucket 2:
- Previous selected (A): Pick 1, from Bucket 1
- Next bucket average (C): avg(Bucket 3 points)
- Keep the point from Bucket 2 (B) that makes the largest triangle

For Bucket 1, A is the first point. The code numbers the buckets from 0, so Bucket 1 in this diagram is `i = 0` in the code below.
```

## TypeScript Implementation

Here is the implementation; save it as `lttb.ts`. It expects points sorted by x. It handles empty arrays and small thresholds, but it does not check for null points or NaN values; the Production Checklist at the end covers that. In the code, `a` is the index of point A (`pointA` is the point itself), `avgX` and `avgY` are point C, and each `data[j]` in the current bucket is a candidate B:

```typescript
interface Point {
  x: number;
  y: number;
}

/**
 * Largest Triangle Three Buckets downsampling algorithm
 * 
 * @param data - Source data points (must be sorted by x)
 * @param threshold - Target number of points in output
 * @returns Downsampled points that preserve visual shape
 */
export function largestTriangleThreeBuckets(
  data: Point[],
  threshold: number
): Point[] {
  const dataLength = data.length;

  // If threshold is greater than or equal to data length, return all data
  if (threshold >= dataLength) {
    return data;
  }

  // With a threshold of 2 or less, return only the first and last points
  if (threshold <= 2) {
    return [data[0], data[dataLength - 1]];
  }

  const sampled: Point[] = [];
  
  // Always include the first point
  sampled.push(data[0]);

  // Bucket size. Leave room for start and endpoints
  const bucketSize = (dataLength - 2) / (threshold - 2);

  // Index of currently selected point in previous bucket
  let a = 0;

  for (let i = 0; i < threshold - 2; i++) {
    // Calculate point average for next bucket (used as point C in triangle)
    let avgX = 0;
    let avgY = 0;

    const avgRangeStart = Math.floor((i + 1) * bucketSize) + 1;
    const avgRangeEnd = Math.min(
      Math.floor((i + 2) * bucketSize) + 1,
      dataLength
    );
    const avgRangeLength = avgRangeEnd - avgRangeStart;

    for (let j = avgRangeStart; j < avgRangeEnd; j++) {
      avgX += data[j].x;
      avgY += data[j].y;
    }
    avgX /= avgRangeLength;
    avgY /= avgRangeLength;

    // Get the range for this bucket
    const rangeStart = Math.floor(i * bucketSize) + 1;
    const rangeEnd = Math.floor((i + 1) * bucketSize) + 1;

    // Point a is the previously selected point
    const pointA = data[a];

    let maxArea = -1;
    let maxAreaPoint = 0;

    // Find point in current bucket that forms largest triangle
    for (let j = rangeStart; j < rangeEnd; j++) {
      // Same area as the formula in "The Triangle Area", rearranged with A as the origin
      const area = Math.abs(
        (pointA.x - avgX) * (data[j].y - pointA.y) -
        (pointA.x - data[j].x) * (avgY - pointA.y)
      ) * 0.5;

      if (area > maxArea) {
        maxArea = area;
        maxAreaPoint = j;
      }
    }

    sampled.push(data[maxAreaPoint]);
    a = maxAreaPoint; // This point is the next "point A"
  }

  // Always include the last point
  sampled.push(data[dataLength - 1]);

  return sampled;
}
```

Bucket ranges start at index 1 because the first point is kept on its own. For the last bucket, the average range is clamped to `dataLength`, so point C is the last point.

### Example Usage

```typescript
// Generate sample time-series data: a sine wave with noise
function generateTimeSeriesData(count: number): Point[] {
  const data: Point[] = [];
  for (let i = 0; i < count; i++) {
    const x = i;
    const y = Math.sin(i / 100) * 50 + Math.random() * 10;
    data.push({ x, y });
  }
  return data;
}

// Original data: 10,000 points
const originalData = generateTimeSeriesData(10000);
console.log(`Original: ${originalData.length} points`);

// Downsample to 500 points
const downsampled = largestTriangleThreeBuckets(originalData, 500);
console.log(`Downsampled: ${downsampled.length} points`);
console.log(`Reduction: ${((1 - downsampled.length / originalData.length) * 100).toFixed(1)}%`);

// Use in your charting library
// chart.setData(downsampled); // Much faster rendering!
```

## Real-World Example: Stock Price Visualization

Say you're building a stock charting application. You fetch intraday data, one price per trading minute for a year: 252 trading days of 390 minutes each, or 98,280 points:

```typescript
interface StockPrice {
  timestamp: number;  // Unix timestamp
  price: number;
  volume: number;
}

function downsampleStockData(
  prices: StockPrice[],
  targetPoints: number
): StockPrice[] {
  // Convert to Point format
  const points: Point[] = prices.map(p => ({
    x: p.timestamp,
    y: p.price
  }));

  // Apply LTTB
  const downsampled = largestTriangleThreeBuckets(points, targetPoints);

  // Match each kept point to its original record by timestamp, to keep volume
  const downsampledPrices: StockPrice[] = [];
  let priceIndex = 0;
  
  for (const point of downsampled) {
    while (priceIndex < prices.length && prices[priceIndex].timestamp !== point.x) {
      priceIndex++;
    }
    if (priceIndex < prices.length) {
      downsampledPrices.push(prices[priceIndex]);
    }
  }

  return downsampledPrices;
}

// Usage: 1 year of trading minutes (98,280 points) down to 2,000
// fetchStockPrices stands in for your own data source
const yearData = await fetchStockPrices('AAPL', '1y', '1m');
const chartData = downsampleStockData(yearData, 2000); // 98.0% reduction
```

LTTB returns plain `Point` objects, so the loop matches each one back to its full record. It assumes that timestamps are unique and sorted.

## Where to Use LTTB

### 1. Time-Series Visualization

**Perfect fit:**
- Financial charts (stocks, crypto, forex)
- Server metrics (CPU, memory, network)
- IoT sensor data (temperature, pressure, vibration)
- Analytics dashboards (user activity, sales trends)

**Why it works:** Time-series data is dense and continuous. LTTB preserves the temporal patterns you need to spot trends and anomalies.

### 2. Real-Time Monitoring Dashboards

When streaming live metrics:

```typescript
class MetricsBuffer {
  private buffer: Point[] = [];
  private maxBufferSize = 100000;
  private displayThreshold = 1000;

  add(point: Point): void {
    this.buffer.push(point);
    
    // Downsample when buffer gets large
    if (this.buffer.length > this.maxBufferSize) {
      this.buffer = largestTriangleThreeBuckets(
        this.buffer,
        this.maxBufferSize / 2
      );
    }
  }

  getDisplayData(): Point[] {
    if (this.buffer.length <= this.displayThreshold) {
      return this.buffer;
    }
    return largestTriangleThreeBuckets(this.buffer, this.displayThreshold);
  }
}
```

When the buffer passes 100,000 points, `add` cuts it to 50,000 with LTTB. The next cut downsamples those already-downsampled points again, so older data loses a little more detail each time. `getDisplayData` then reduces the buffer to 1,000 points for the chart.

### 3. Historical Data Analysis

When users zoom out to see long time ranges:

```typescript
function adaptiveDownsample(
  data: Point[],
  viewportWidth: number // in pixels
): Point[] {
  // One point per 2 pixels is enough detail for a line chart
  const targetPoints = Math.floor(viewportWidth / 2);
  
  // Skip downsampling when the data is at most twice the target
  if (data.length <= targetPoints * 2) {
    return data;
  }
  
  return largestTriangleThreeBuckets(data, targetPoints);
}

// When user zooms in/out, re-downsample for current view
// chart, allData and getDataInViewport come from your charting setup
chart.on('zoom', (viewport) => {
  const visible = getDataInViewport(allData, viewport);
  const downsampled = adaptiveDownsample(visible, viewport.width);
  chart.update(downsampled);
});
```

### 4. Data Export/Transfer

Reduce payload size for client applications:

```typescript
// API endpoint that returns downsampled data
// Express-style handler; db is your own data layer
app.get('/api/metrics/:id', async (req, res) => {
  const requested = Number(req.query.points ?? 1000);
  const threshold = Number.isFinite(requested) ? Math.min(requested, 5000) : 1000;
  const fullData = await db.getMetrics(req.params.id);
  
  // Downsample before sending
  const optimized = largestTriangleThreeBuckets(
    fullData,
    threshold // capped at 5,000 to bound the response size
  );
  
  res.json({
    original_count: fullData.length,
    downsampled_count: optimized.length,
    data: optimized
  });
});
```

## Where NOT to Use LTTB

### 1. Statistical Analysis

**Don't use for:**
- Calculating averages, medians, or percentiles
- Standard deviation or variance
- Correlation analysis
- Any statistical computation

**Why:** LTTB biases its picks toward extremes and changes, which skews statistics.

```typescript
// BAD: Statistical analysis on downsampled data
const downsampled = largestTriangleThreeBuckets(data, 1000);
const biasedAverage = downsampled.reduce((s, p) => s + p.y, 0) / downsampled.length;
// ❌ This average is WRONG - biased toward peaks/valleys

// GOOD: Calculate stats on full data
const average = data.reduce((s, p) => s + p.y, 0) / data.length;
// ✅ Correct statistical average
```

### 2. Scientific Precision Requirements

**Don't use for:**
- Medical device data where every reading matters
- Financial audit trails (compliance requires complete data)
- Scientific experiments requiring exact measurements
- Legal or regulatory data (must be complete and unaltered)

**Why:** Downsampling discards information. If you need provable accuracy or regulatory compliance, you can't afford to lose any data points.

### 3. Sparse or Non-Continuous Data

**Don't use for:**
- Event logs (sparse, discrete events)
- Transaction records (each record is unique)
- Categorical data (non-numeric or discrete categories)
- Already small datasets (< 1000 points), which a chart can draw in full

**Example of bad fit:**

```typescript
// BAD: Transaction log data
const transactions = [
  { time: 100, type: 'purchase', amount: 50 },
  { time: 500, type: 'refund', amount: 20 },
  { time: 1000, type: 'purchase', amount: 100 }
];
// Each transaction is important - can't downsample without losing critical info

// BAD: Sparse sensor data with gaps
const sensorData = [
  { x: 0, y: 10 },
  { x: 1000, y: 12 },  // 1000 time units gap
  { x: 1001, y: 50 }   // Sudden spike
];
// In a long series with gaps like this, buckets hold equal numbers of points, not equal
// spans of time, so one bucket can span a long gap and a dense burst at once and keep one point
```

### 4. When You Need Exact Points

**Don't use for:**
- Min/Max calculations (LTTB can drop the exact extreme, as the example below shows)
- Exact zero-crossing detection
- Precise integration or area-under-curve
- Anomaly detection algorithms (run them on the full data; LTTB helps you see anomalies on a chart, not find them)

LTTB may miss the exact maximum if another point in the same bucket forms a larger triangle:

```typescript
// Missing the exact peak
const data = [
  { x: 0, y: 10 },
  { x: 1, y: 11 },
  { x: 2, y: 100 },  // Peak
  { x: 3, y: -120 }, // Deeper dip in the same bucket
  { x: 4, y: 10 }
];

const downsampled = largestTriangleThreeBuckets(data, 3);
// Threshold 3 keeps the first and last points, and x = 1, 2, 3 form the one bucket
// [{ x: 0, y: 10 }, { x: 3, y: -120 }, { x: 4, y: 10 }]
// The dip forms a larger triangle than the peak, so the maximum (y = 100) is dropped
// To guarantee the peak, keep each bucket's maximum instead (max aggregation)
```

## Trade-offs and Limitations

### Memory vs. Visual Accuracy

The threshold parameter is your knob. A higher threshold keeps more points, which gives better visual accuracy but slower rendering and more memory use.

**Starting points by use case** (tune them on your own charts):
- **Live monitoring:** 500-1,000 points, for a chart that redraws every second
- **Historical analysis:** 1,000-2,000 points, recomputed when the user changes the time range
- **Export/reporting:** 2,000-5,000 points, computed once per report

### Processing Cost

The O(n) scan isn't free. This benchmark uses `generateTimeSeriesData` from "Example Usage":

```typescript
// Benchmark on Apple M4, Node v24.18.1: one warm-up run, then the median of 5 runs
const sizes = [10_000, 100_000, 1_000_000, 10_000_000];
sizes.forEach(size => {
  const data = generateTimeSeriesData(size);
  largestTriangleThreeBuckets(data, 1000);
  const runs: number[] = [];
  for (let k = 0; k < 5; k++) {
    const start = performance.now();
    largestTriangleThreeBuckets(data, 1000);
    runs.push(performance.now() - start);
  }
  const median = runs.sort((a, b) => a - b)[2];
  console.log(`${size.toLocaleString()} points: ${median.toFixed(2)}ms`);
});

// Results:
// 10,000 points: 0.29ms
// 100,000 points: 0.79ms
// 1,000,000 points: 3.43ms
// 10,000,000 points: 44.07ms
```

For 10M points, 44ms is longer than the 16ms a browser has to draw one frame at 60 fps, so running it on the main thread drops frames. Three ways to keep the page responsive:
- **Pre-compute:** Downsample on the backend and cache the results
- **Web Workers:** Run LTTB on a background thread
- **Progressive loading:** Draw a low-threshold downsample first, then compute the full one when the browser is idle (for example with `requestIdleCallback`)

`postMessage` copies whatever you send to a worker. In Node v24.18.1 on the same Apple M4, `structuredClone`, the copy that `postMessage` makes, took about 4 seconds for the 10M point objects from `generateTimeSeriesData(10_000_000)` (median of 3 runs), far more than the 44ms that LTTB needs. So load the data inside the worker and send back only the downsampled points. The example assumes `/data/cpu-full.json` holds an array of `{ x, y }` points. The fetch and JSON parse still take time, but they run in the worker, off the main thread:

```typescript
// main.ts
const worker = new Worker(new URL('./lttb-worker.ts', import.meta.url), { type: 'module' });
worker.postMessage({ url: '/data/cpu-full.json', threshold: 1000 });
worker.onmessage = (e) => {
  chart.setData(e.data.downsampled);
};

// lttb-worker.ts
import { largestTriangleThreeBuckets } from './lttb';

self.onmessage = async (e) => {
  const { url, threshold } = e.data;
  const data = await (await fetch(url)).json();
  const downsampled = largestTriangleThreeBuckets(data, threshold);
  self.postMessage({ downsampled });
};
```

### Loss of Statistical Properties

The biggest limitation is that downsampled data looks right on a chart and gives wrong numbers in calculations. Here is the effect on 1,000,000 points from `generateTimeSeriesData` (in "Example Usage"), downsampled to 1,000. The noise is random, so the numbers move slightly from run to run:

| Metric | Full Data (1M pts) | LTTB (1000pts) | Error vs full data |
|--------|--------------------|----------------|-------|
| Mean | 5.011 | 5.095 | +1.7% |
| Std Dev | 35.47 | 53.89 | +51.9% |
| Min | -50.00 | -49.99 | 0.0% |
| Max | 59.99 | 59.99 | 0.0% |

**Why:** LTTB favors extremes. Here each bucket of 1,000 points spans more than one full sine wave (one wave is about 628 points), so LTTB keeps points near the peaks and troughs, which sit far from the mean, and the standard deviation grows by about half. The minimum and maximum survive in this run, but LTTB does not guarantee that: the example under "When You Need Exact Points" shows it dropping the maximum.

**Solution:** keep two views of the data: the downsampled points for charts and the full data for calculations:

```typescript
interface DatasetView {
  visual: Point[];    // LTTB downsampled for charts
  stats: Point[];     // Full data for calculations
}

function prepareDataset(raw: Point[], threshold: number): DatasetView {
  return {
    visual: largestTriangleThreeBuckets(raw, threshold),
    stats: raw // Keep full data for accurate calculations
  };
}

// Use visual for rendering, stats for calculations
const dataset = prepareDataset(rawData, 1000);
chart.render(dataset.visual);
const average = dataset.stats.reduce((s, p) => s + p.y, 0) / dataset.stats.length;
```

## Advanced Patterns

### Multi-Resolution Storage

Store multiple resolutions (the full data plus 10,000, 1,000 and 100 points) for different zoom levels:

```typescript
interface MultiResData {
  full: Point[];
  res_10k: Point[];
  res_1k: Point[];
  res_100: Point[];
}

function createMultiResolution(data: Point[]): MultiResData {
  return {
    full: data,
    res_10k: data.length > 10_000 
      ? largestTriangleThreeBuckets(data, 10_000) 
      : data,
    res_1k: largestTriangleThreeBuckets(data, 1_000),
    res_100: largestTriangleThreeBuckets(data, 100)
  };
}

// visibleFraction: the share of the full time range on screen, from 0 to 1
function selectResolution(
  multi: MultiResData,
  visibleFraction: number,
  maxDrawnPoints = 5_000
): Point[] {
  const finestFirst = [multi.full, multi.res_10k, multi.res_1k, multi.res_100];
  return finestFirst.find(level => level.length * visibleFraction <= maxDrawnPoints) ?? multi.res_100;
}
```

`selectResolution` picks the finest level that still puts at most 5,000 points on screen, assuming the points are spread evenly over time. Zoomed out, you get a coarse level; zoomed in far enough, you get the raw data. Draw only the part of that level inside the viewport.

### Streaming with a Sliding Window

For real-time data, keep a sliding window of the latest 10,000 points and downsample only that window. While the window fills, the class below recomputes every 100 points. After the window is full, it drops the oldest point and reruns LTTB over the whole window on every new point:

```typescript
class StreamingLTTB {
  private buffer: Point[] = [];
  private downsampled: Point[] = [];
  private windowSize = 10_000;
  private targetSize = 1_000;

  add(point: Point): void {
    this.buffer.push(point);

    if (this.buffer.length > this.windowSize) {
      this.buffer.shift();
      this.recompute();
    } else if (this.buffer.length % 100 === 0) {
      this.recompute();
    }
  }

  private recompute(): void {
    this.downsampled = largestTriangleThreeBuckets(
      this.buffer,
      this.targetSize
    );
  }

  getView(): Point[] {
    return this.downsampled;
  }
}
```

Before the first 100 points arrive, `getView()` returns an empty array, and between recomputes it returns the last result. Once the window is full, every new point costs one LTTB pass over 10,000 points (0.29ms on an Apple M4, from the benchmark above) plus a `shift()`, which is O(n) on an array. The bucket boundaries move as the window slides, so the kept points can change between recomputes and the line may shift slightly from one update to the next. Unlike `MetricsBuffer`, which keeps a downsampled history of everything, this class shows only the latest 10,000 points.

## Comparison with Other Downsampling Methods

| Method | Pros | Cons | Use Case |
|--------|------|------|----------|
| **Every Nth** | Simplest, O(M) for M kept points | Misses peaks/valleys | Never for charts (use LTTB instead) |
| **Random Sampling** | Unbiased estimates of stats | Unpredictable visual | Estimating statistics from a sample |
| **Min-Max** | Preserves extremes | Loses smooth curves | Range-bound displays |
| **LTTB** | Visual fidelity, O(n) | Stats biased | Time-series charts |
| **M4** (an algorithm, not the Apple chip; Jugel et al., VLDB 2014) | Very fast, keeps extremes | Tied to the chart's pixel width | High-frequency data |

Min-Max keeps the smallest and largest value in each bucket. M4 splits the data into one bucket per pixel column of the chart and keeps four points from each: the first, the last, the minimum and the maximum.

**When to choose LTTB:**
- You're rendering line charts
- Visual shape matters more than exact statistics
- Data is time-ordered and continuous
- You need consistent, reproducible results

## Production Checklist

Before deploying LTTB in production:

- [ ] Verify input data is sorted by x-axis (time)
- [ ] Add tests for empty arrays, single points and threshold >= data length, so a future change to the function cannot break them
- [ ] Add input validation: non-null points, valid numbers
- [ ] Consider pre-computing and caching downsampled views
- [ ] Use Web Workers when one run takes longer than a 16ms frame on your users' devices (on an Apple M4, that happens between 1M and 10M points)
- [ ] Keep full data for statistical calculations
- [ ] Document that downsampled data is for visualization only
- [ ] Log how long each downsample takes, so you see when data growth pushes it past a 16ms frame
- [ ] Test with real production data shapes (spikes, flat lines, noise)

## Conclusion

LTTB is a visualization optimization rather than a general-purpose compression algorithm. It trades statistical accuracy for visual fidelity, and when you're rendering charts that trade is worth making: your users see the overall shape of 1,000,000 points in 1,000, and the browser stays responsive. Draw with the downsampled points and keep the full data for calculations.

The whole implementation fits in about 80 lines of TypeScript, and you can see it running in the [live demo](/demos/lttb/).
