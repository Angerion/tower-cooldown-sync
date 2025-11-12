<script>
  import TimerManager from './lib/TimerManager.svelte'
  import Timeline from './lib/Timeline.svelte'

  let timers = $state([
    { id: 1, name: "Black Hole", cooldown: 60, duration: 32, offset: 0 },
    { id: 2, name: "Chrono Field", cooldown: 120, duration: 36, offset: 30 },
    { id: 3, name: "Golden Tower", cooldown: 120, duration: 50, offset: 0 }
  ]);

  let nextId = $state(4);
  let customTimelineLength = $state(600);

  function addTimer() {
    const newId = nextId++;
    timers = [...timers, { 
      id: newId, 
      name: `Timer ${newId}`, 
      cooldown: 60, 
      duration: 30,
      offset: 0
    }];
  }
  
  function updateTimelineLength(length) {
    customTimelineLength = length;
  }

  function removeTimer(id) {
    timers = timers.filter(t => t.id !== id);
  }

  function updateTimer(id, field, value) {
    timers = timers.map(t => 
      t.id === id ? { ...t, [field]: value } : t
    );
  }
</script>

<main>
  <h1>Tower Cooldown Sync</h1>
  <p class="subtitle">Visualize buff durations and overlapping</p>

  <div class="container">
    <div class="controls">
      <h2>Timers</h2>
      <TimerManager 
        {timers} 
        {addTimer}
        {removeTimer}
        {updateTimer}
      />
    </div>

    <div class="visualization">
      <h2>Timeline Visualization</h2>
      <Timeline {timers} {customTimelineLength} {updateTimelineLength} />
    </div>
  </div>
</main>

<style>
  main {
    padding: 2rem;
    max-width: 1400px;
    margin: 0 auto;
  }

  h1 {
    text-align: center;
    margin-bottom: 0.5rem;
  }

  .subtitle {
    text-align: center;
    color: #888;
    margin-bottom: 2rem;
  }

  .container {
    display: grid;
    grid-template-columns: 1fr;
    gap: 2rem;
    align-items: start;
  }

  .controls h2,
  .visualization h2 {
    margin-top: 0;
    margin-bottom: 1rem;
    font-size: 1.5rem;
  }
</style>
