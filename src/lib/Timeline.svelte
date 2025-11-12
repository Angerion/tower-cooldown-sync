<script>
  let { timers, customTimelineLength, updateTimelineLength } = $props();
  
  // Calculate the least common multiple for visualization
  function gcd(a, b) {
    a = Math.abs(a);
    b = Math.abs(b);
    while (b !== 0) {
      const temp = b;
      b = a % b;
      a = temp;
    }
    return a;
  }
  
  function lcm(a, b) {
    return (a * b) / gcd(a, b);
  }
  
  // Calculate timeline parameters
  let timelineLength = $derived.by(() => {
    if (customTimelineLength !== null && customTimelineLength > 0) {
      return customTimelineLength;
    }
    
    if (timers.length === 0) return 120;
    
    // Calculate LCM of all cooldowns for a complete cycle
    let cooldownLcm = timers.reduce((acc, t) => lcm(acc, t.cooldown || 1), 1);
    // Cap at reasonable max
    return Math.min(cooldownLcm, 300);
  });
  
  // Generate timeline events for each timer
  function getTimelineEvents(timer, maxTime) {
    const events = [];
    let currentTime = timer.offset || 0;
    
    while (currentTime < maxTime) {
      events.push({
        start: currentTime,
        end: currentTime + timer.duration,
        timerId: timer.id,
        timerName: timer.name
      });
      currentTime += timer.cooldown;
    }
    
    return events;
  }
  
  // Find all overlapping regions with timer combinations
  function findAllOverlaps(events, timers) {
    const allOverlaps = {};
    
    // For each unique combination of timers
    const timerIds = timers.map(t => t.id);
    
    // Generate all combinations (2 or more timers)
    for (let size = 2; size <= timerIds.length; size++) {
      const combinations = getCombinations(timerIds, size);
      
      for (const combo of combinations) {
        const comboKey = combo.sort((a, b) => a - b).join(',');
        allOverlaps[comboKey] = findOverlapForTimers(events, combo);
      }
    }
    
    return allOverlaps;
  }
  
  // Generate combinations of array elements
  function getCombinations(arr, size) {
    if (size === 1) return arr.map(x => [x]);
    if (size > arr.length) return [];
    
    const result = [];
    
    function combine(start, combo) {
      if (combo.length === size) {
        result.push([...combo]);
        return;
      }
      
      for (let i = start; i < arr.length; i++) {
        combo.push(arr[i]);
        combine(i + 1, combo);
        combo.pop();
      }
    }
    
    combine(0, []);
    return result;
  }
  
  // Find overlap periods for a specific set of timers
  function findOverlapForTimers(events, timerIds) {
    const overlaps = [];
    
    // Get events for these specific timers
    const relevantEvents = events.filter(e => timerIds.includes(e.timerId));
    
    // For each time point, check if all timers in the combination are active
    const timePoints = new Set();
    relevantEvents.forEach(e => {
      timePoints.add(e.start);
      timePoints.add(e.end);
    });
    
    const sortedPoints = Array.from(timePoints).sort((a, b) => a - b);
    
    for (let i = 0; i < sortedPoints.length - 1; i++) {
      const start = sortedPoints[i];
      const end = sortedPoints[i + 1];
      const midPoint = (start + end) / 2;
      
      // Check if all timers in this combination are active at midpoint
      const activeTimers = new Set(
        relevantEvents
          .filter(e => e.start <= midPoint && e.end > midPoint)
          .map(e => e.timerId)
      );
      
      if (timerIds.every(id => activeTimers.has(id))) {
        overlaps.push({ start, end, timerIds: [...timerIds] });
      }
    }
    
    return overlaps;
  }
  
  // Merge overlapping intervals
  function mergeIntervals(intervals) {
    if (!intervals || intervals.length === 0) return [];
    
    // Sort intervals by start time
    const sorted = intervals
      .map(({start, end}) => ({start, end}))
      .sort((a, b) => a.start - b.start);
    
    const merged = [];
    let prev = sorted[0];
    
    for (let i = 1; i < sorted.length; i++) {
      const curr = sorted[i];
      if (curr.start <= prev.end) {
        // Overlapping, merge
        prev.end = Math.max(prev.end, curr.end);
      } else {
        merged.push(prev);
        prev = curr;
      }
    }
    merged.push(prev);
    return merged;
  }
  
  let allEvents = $derived(timers.flatMap(timer => getTimelineEvents(timer, timelineLength)));
  let allOverlapsByCombo = $derived(findAllOverlaps(allEvents, timers));
  
  // Get total overlap (all timers active at once)
  let totalOverlapKey = $derived(timers.map(t => t.id).sort((a, b) => a - b).join(','));
  let totalOverlapRegions = $derived(
    timers.length >= 2 && allOverlapsByCombo[totalOverlapKey] 
      ? allOverlapsByCombo[totalOverlapKey] 
      : []
  );
  
  // Special overlap: Black Hole OR Chrono Field (at least one active)
  let bhOrCfOverlap = $derived.by(() => {
    const bhTimer = timers.find(t => t.name === "Black Hole");
    const cfTimer = timers.find(t => t.name === "Chrono Field");
    
    if (!bhTimer || !cfTimer) return [];
    
    const bhEvents = getTimelineEvents(bhTimer, timelineLength);
    const cfEvents = getTimelineEvents(cfTimer, timelineLength);
    
    // Merge all BH and CF events into a single timeline
    const allBhCfEvents = [...bhEvents, ...cfEvents].sort((a, b) => a.start - b.start);
    
    // Merge overlapping/adjacent intervals
    const merged = [];
    if (allBhCfEvents.length === 0) return [];
    
    let current = { start: allBhCfEvents[0].start, end: allBhCfEvents[0].end };
    
    for (let i = 1; i < allBhCfEvents.length; i++) {
      const event = allBhCfEvents[i];
      if (event.start <= current.end) {
        // Overlapping or adjacent, merge
        current.end = Math.max(current.end, event.end);
      } else {
        // No overlap, push current and start new
        merged.push({ ...current, timerIds: [bhTimer.id, cfTimer.id] });
        current = { start: event.start, end: event.end };
      }
    }
    merged.push({ ...current, timerIds: [bhTimer.id, cfTimer.id] });
    
    return merged;
  });
  
  // Calculate overlap statistics
  let totalOverlapDuration = $derived(
    totalOverlapRegions.reduce((sum, overlap) => 
      sum + (overlap.end - overlap.start), 0
    )
  );
  let overlapPercentage = $derived(
    timelineLength > 0 
      ? ((totalOverlapDuration / timelineLength) * 100).toFixed(1)
      : 0
  );
  
  // Calculate uptime metrics for each timer
  function calculateTimerMetrics(timer, allEvents, maxTime) {
    const timerEvents = getTimelineEvents(timer, maxTime);
    
    // Total active time for this timer
    const totalActiveTime = timerEvents.reduce((sum, event) => 
      sum + Math.min(event.end, maxTime) - event.start, 0
    );
    
    // Time when timer is active alone (not overlapping with others)
    let aloneTime = 0;
    for (const event of timerEvents) {
      const eventStart = event.start;
      const eventEnd = Math.min(event.end, maxTime);
      
      // Check at 0.1 second intervals of this event
      const steps = Math.floor((eventEnd - eventStart) * 10);
      for (let i = 0; i < steps; i++) {
        const t = eventStart + (i * 0.1);
        const hasOverlap = allEvents.some(e => 
          e.timerId !== timer.id && 
          e.start <= t && 
          e.end > t
        );
        if (!hasOverlap) {
          aloneTime += 0.1;
        }
      }
    }
    
    // Time when timer overlaps with at least one other timer
    const overlapTime = totalActiveTime - aloneTime;
    
    // Uptime percentages
    const uptimePercentage = (totalActiveTime / maxTime * 100).toFixed(1);
    const alonePercentage = (aloneTime / maxTime * 100).toFixed(1);
    const overlapEffectiveness = totalActiveTime > 0 
      ? (overlapTime / totalActiveTime * 100).toFixed(1)
      : 0;
    
    return {
      totalActiveTime: totalActiveTime.toFixed(1),
      uptimePercentage,
      aloneTime: aloneTime.toFixed(1),
      alonePercentage,
      overlapTime: overlapTime.toFixed(1),
      overlapEffectiveness
    };
  }
  
  let timerMetrics = $derived(timers.map(timer => ({
    ...timer,
    metrics: calculateTimerMetrics(timer, allEvents, timelineLength)
  })));
  
  // Color palette for different timers
  const colors = [
    '#646cff', '#ff3e3e', '#4ade80', '#facc15', '#a78bfa', 
    '#fb923c', '#ec4899', '#14b8a6', '#f87171', '#34d399'
  ];
  
  function getTimerColor(timerId) {
    return colors[(timerId - 1) % colors.length];
  }
  
  // Get combination label
  function getComboLabel(timerIds) {
    return timerIds
      .map(id => timers.find(t => t.id === id)?.name || `Timer ${id}`)
      .join(' + ');
  }
</script>

<div class="timeline-container">
  <div class="stats">
    <div class="stat-item">
      <span class="stat-label">Timeline Duration:</span>
      <div class="stat-value-editable">
        <input
          type="number"
          value={timelineLength}
          oninput={(e) => updateTimelineLength(Math.max(10, parseInt(e.target.value) || 120))}
          min="10"
          class="timeline-input"
        />
        <span>s</span>
      </div>
    </div>
    <div class="stat-item">
      <span class="stat-label">Total Overlap Time:</span>
      <span class="stat-value">{totalOverlapDuration.toFixed(1)}s ({overlapPercentage}%)</span>
    </div>
    <div class="stat-item">
      <span class="stat-label">Overlap Count:</span>
      <span class="stat-value">{totalOverlapRegions.length}</span>
    </div>
  </div>
  
  {#if timers.length > 0}
    <div class="timer-metrics">
      <h3>Timer Metrics</h3>
      <div class="metrics-grid">
        {#each timerMetrics as timer (timer.id)}
          <div class="metric-card" style="border-left: 4px solid {getTimerColor(timer.id)}">
            <div class="metric-header" style="color: {getTimerColor(timer.id)}">
              {timer.name}
            </div>
            <div class="metric-rows">
              <div class="metric-row">
                <span class="metric-label">Total Uptime:</span>
                <span class="metric-val">{timer.metrics.totalActiveTime}s ({timer.metrics.uptimePercentage}%)</span>
              </div>
              <div class="metric-row">
                <span class="metric-label">Alone Time:</span>
                <span class="metric-val">{timer.metrics.aloneTime}s ({timer.metrics.alonePercentage}%)</span>
              </div>
              <div class="metric-row">
                <span class="metric-label">Overlap Time:</span>
                <span class="metric-val">{timer.metrics.overlapTime}s</span>
              </div>
              <div class="metric-row">
                <span class="metric-label">Overlap Effectiveness:</span>
                <span class="metric-val highlight">{timer.metrics.overlapEffectiveness}%</span>
              </div>
            </div>
          </div>
        {/each}
      </div>
    </div>
  {/if}
  
  <div class="timeline">
    <div class="time-ruler">
      {#each Array(Math.ceil(timelineLength / 10)) as _, i}
        <div class="time-marker" style="left: {(i * 10 / timelineLength) * 100}%">
          {i * 10}s
        </div>
      {/each}
    </div>
    
    {#each timers as timer (timer.id)}
      <div class="timer-row">
        <div class="timer-label" style="color: {getTimerColor(timer.id)}">
          {timer.name}
        </div>
        <div class="timer-track">
          {#each getTimelineEvents(timer, timelineLength) as event}
            <div 
              class="timer-event"
              role="img"
              aria-label="{timer.name} active from {event.start}s to {event.end}s"
              style="
                left: {(event.start / timelineLength) * 100}%;
                width: {((event.end - event.start) / timelineLength) * 100}%;
                background-color: {getTimerColor(timer.id)};
              "
              title="{timer.name}: {event.start}s - {event.end}s"
            >
            </div>
          {/each}
        </div>
      </div>
    {/each}
    
    {#if timers.length >= 2}
      {#each Object.entries(allOverlapsByCombo).sort((a, b) => b[0].split(',').length - a[0].split(',').length) as [comboKey, regions]}
        {#if regions.length > 0}
          <div class="timer-row overlap-row">
            <div class="timer-label overlap-label">
              {getComboLabel(comboKey.split(',').map(Number))}
            </div>
            <div class="timer-track">
              {#each regions as overlap}
                <div 
                  class="overlap-event"
                  role="img"
                  aria-label="Overlap from {overlap.start}s to {overlap.end}s"
                  style="
                    left: {(overlap.start / timelineLength) * 100}%;
                    width: {((overlap.end - overlap.start) / timelineLength) * 100}%;
                  "
                  title="{getComboLabel(overlap.timerIds)}: {overlap.start}s - {overlap.end}s"
                >
                </div>
              {/each}
            </div>
          </div>
        {/if}
      {/each}
      
      {#if bhOrCfOverlap.length > 0}
        <div class="timer-row overlap-row special-overlap-row">
          <div class="timer-label special-overlap-label">
            BH or CF (Either Active)
          </div>
          <div class="timer-track">
            {#each bhOrCfOverlap as overlap}
              <div 
                class="special-overlap-event"
                role="img"
                aria-label="Black Hole or Chrono Field active from {overlap.start}s to {overlap.end}s"
                style="
                  left: {(overlap.start / timelineLength) * 100}%;
                  width: {((overlap.end - overlap.start) / timelineLength) * 100}%;
                "
                title="BH or CF: {overlap.start}s - {overlap.end}s"
              >
              </div>
            {/each}
          </div>
        </div>
      {/if}
    {/if}
  </div>
  
  {#if timers.length === 0}
    <div class="empty-state">
      Add timers to see the visualization
    </div>
  {/if}
</div>

<style>
  .timeline-container {
    background: #2a2a2a;
    border-radius: 8px;
    padding: 1.5rem;
    border: 1px solid #444;
  }

  .stats {
    display: flex;
    gap: 2rem;
    margin-bottom: 1.5rem;
    flex-wrap: wrap;
  }

  .stat-item {
    display: flex;
    flex-direction: column;
    gap: 0.25rem;
  }

  .stat-label {
    font-size: 0.875rem;
    color: #aaa;
  }

  .stat-value {
    font-size: 1.25rem;
    font-weight: 600;
    color: #646cff;
  }

  .stat-value-editable {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    font-size: 1.25rem;
    font-weight: 600;
    color: #646cff;
  }

  .timeline-input {
    width: 80px;
    padding: 0.25rem 0.5rem;
    border-radius: 4px;
    border: 1px solid #444;
    background: #1a1a1a;
    color: #646cff;
    font-size: 1.25rem;
    font-weight: 600;
  }

  .timeline-input:focus {
    outline: none;
    border-color: #646cff;
  }

  .timer-metrics {
    margin-bottom: 1.5rem;
  }

  .timer-metrics h3 {
    margin: 0 0 1rem 0;
    font-size: 1.1rem;
    color: #aaa;
  }

  .metrics-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
    gap: 1rem;
    margin-bottom: 1.5rem;
  }

  .metric-card {
    background: #1a1a1a;
    padding: 1rem;
    border-radius: 6px;
  }

  .metric-header {
    font-weight: 600;
    font-size: 1rem;
    margin-bottom: 0.75rem;
  }

  .metric-rows {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
  }

  .metric-row {
    display: flex;
    justify-content: space-between;
    font-size: 0.875rem;
  }

  .metric-label {
    color: #aaa;
  }

  .metric-val {
    font-weight: 600;
    color: #fff;
  }

  .metric-val.highlight {
    color: #4ade80;
  }

  .timeline {
    position: relative;
    padding-top: 2rem;
  }

  .time-ruler {
    position: absolute;
    top: 0;
    left: 150px;
    right: 0;
    height: 2rem;
    border-bottom: 1px solid #444;
  }

  .time-marker {
    position: absolute;
    font-size: 0.75rem;
    color: #888;
    transform: translateX(-50%);
  }

  .timer-row {
    display: flex;
    align-items: center;
    margin-bottom: 0.75rem;
    min-height: 40px;
  }

  .timer-label {
    width: 150px;
    font-weight: 600;
    padding-right: 1rem;
    flex-shrink: 0;
  }

  .overlap-label {
    color: #facc15;
  }

  .timer-track {
    flex: 1;
    height: 32px;
    background: #1a1a1a;
    border-radius: 4px;
    position: relative;
    border: 1px solid #333;
  }

  .timer-event {
    position: absolute;
    top: 0;
    height: 100%;
    border-radius: 4px;
    opacity: 0.8;
    transition: opacity 0.2s;
    border: 1px solid rgba(255, 255, 255, 0.2);
  }

  .timer-event:hover {
    opacity: 1;
    z-index: 10;
  }

  .overlap-event {
    position: absolute;
    top: 0;
    height: 100%;
    background: #facc15;
    border-radius: 4px;
    opacity: 0.7;
    border: 2px solid #fbbf24;
  }

  .overlap-event:hover {
    opacity: 0.9;
  }

  .overlap-row {
    margin-top: 1.5rem;
    padding-top: 1rem;
    border-top: 1px solid #444;
  }

  .special-overlap-row {
    margin-top: 1rem;
    padding-top: 1rem;
    border-top: 2px dashed #a78bfa;
  }

  .special-overlap-label {
    color: #a78bfa;
    font-style: italic;
  }

  .special-overlap-event {
    position: absolute;
    top: 0;
    height: 100%;
    background: #a78bfa;
    border-radius: 4px;
    opacity: 0.6;
    border: 2px solid #9333ea;
  }

  .special-overlap-event:hover {
    opacity: 0.85;
  }

  .empty-state {
    text-align: center;
    padding: 3rem;
    color: #888;
    font-style: italic;
  }

  @media (prefers-color-scheme: light) {
    .timeline-container {
      background: #f5f5f5;
      border-color: #ddd;
    }

    .time-ruler {
      border-bottom-color: #ddd;
    }

    .stat-label {
      color: #666;
    }

    .timeline-input {
      background: white;
      border-color: #ddd;
    }

    .timer-metrics h3 {
      color: #666;
    }

    .metric-card {
      background: white;
    }

    .metric-label {
      color: #666;
    }

    .metric-val {
      color: #000;
    }

    .timer-track {
      background: white;
      border-color: #ddd;
    }

    .overlap-row {
      border-top-color: #ddd;
    }

    .empty-state {
      color: #666;
    }
  }
</style>
