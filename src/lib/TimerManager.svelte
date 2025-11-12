<script>
  let { timers, addTimer, removeTimer, updateTimer } = $props();
</script>

<div class="timer-manager">
  {#each timers as timer (timer.id)}
    <div class="timer-card">
      <div class="timer-header">
        <input
          type="text"
          value={timer.name}
          oninput={(e) => updateTimer(timer.id, 'name', e.target.value)}
          class="timer-name-input"
        />
        <button
          onclick={() => removeTimer(timer.id)}
          class="remove-btn"
          title="Remove timer"
        >
          ×
        </button>
      </div>
      
      <div class="timer-controls">
        <div class="control-group">
          <label for="cooldown-{timer.id}">Cooldown (s)</label>
          <input
            id="cooldown-{timer.id}"
            type="number"
            value={timer.cooldown}
            oninput={(e) => updateTimer(timer.id, 'cooldown', Math.max(1, parseInt(e.target.value) || 1))}
            min="1"
          />
        </div>
        
        <div class="control-group">
          <label for="duration-{timer.id}">Duration (s)</label>
          <input
            id="duration-{timer.id}"
            type="number"
            value={timer.duration}
            oninput={(e) => updateTimer(timer.id, 'duration', Math.max(1, parseInt(e.target.value) || 1))}
            min="1"
          />
        </div>
        
        <div class="control-group">
          <label for="offset-{timer.id}">Start Offset (s)</label>
          <input
            id="offset-{timer.id}"
            type="number"
            value={timer.offset}
            oninput={(e) => updateTimer(timer.id, 'offset', Math.max(0, parseInt(e.target.value) || 0))}
            min="0"
          />
        </div>
      </div>
    </div>
  {/each}
  
  <button onclick={addTimer} class="add-timer-btn">
    + Add Timer
  </button>
</div>

<style>
  .timer-manager {
    display: flex;
    flex-direction: column;
    gap: 1rem;
  }

  .timer-card {
    background: #2a2a2a;
    border-radius: 8px;
    padding: 1rem;
    border: 1px solid #444;
  }

  .timer-header {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    margin-bottom: 1rem;
  }

  .timer-name-input {
    flex: 1;
    font-size: 1.1rem;
    font-weight: 600;
    background: transparent;
    border: none;
    border-bottom: 2px solid transparent;
    color: inherit;
    padding: 0.25rem 0;
    transition: border-color 0.2s;
  }

  .timer-name-input:focus {
    outline: none;
    border-bottom-color: #646cff;
  }

  .remove-btn {
    background: #ff3e3e;
    color: white;
    border: none;
    border-radius: 50%;
    width: 28px;
    height: 28px;
    font-size: 1.5rem;
    line-height: 1;
    cursor: pointer;
    padding: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: background 0.2s;
  }

  .remove-btn:hover {
    background: #ff5555;
  }

  .timer-controls {
    display: grid;
    grid-template-columns: 1fr 1fr 1fr;
    gap: 1rem;
  }

  .control-group {
    display: flex;
    flex-direction: column;
    gap: 0.25rem;
  }

  .control-group label {
    font-size: 0.875rem;
    color: #aaa;
  }

  .control-group input {
    padding: 0.5rem;
    border-radius: 4px;
    border: 1px solid #444;
    background: #1a1a1a;
    color: inherit;
    font-size: 1rem;
  }

  .control-group input:focus {
    outline: none;
    border-color: #646cff;
  }

  .add-timer-btn {
    width: 100%;
    padding: 0.75rem;
    background: #646cff;
    color: white;
    border: none;
    border-radius: 8px;
    font-size: 1rem;
    font-weight: 600;
    cursor: pointer;
    transition: background 0.2s;
  }

  .add-timer-btn:hover {
    background: #535bf2;
  }

  @media (prefers-color-scheme: light) {
    .timer-card {
      background: #f5f5f5;
      border-color: #ddd;
    }

    .control-group label {
      color: #666;
    }

    .control-group input {
      background: white;
      border-color: #ddd;
    }
  }
</style>
