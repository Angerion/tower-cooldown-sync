# Tower Cooldown Sync

A Svelte application to visualize customizable buff durations and their overlapping periods.

## Features

- **Customizable Timers**: Add, remove, and configure multiple timers with custom names, cooldowns, and durations
- **Visual Timeline**: Interactive timeline showing when each buff is active
- **Overlap Detection**: Automatically highlights overlapping periods between different buffs
- **Detailed Metrics**: 
  - Total uptime for each timer
  - Alone time (when buff is active without overlap)
  - Overlap time and effectiveness percentage
  - Overall timeline statistics

## Default Timers

The app comes with two default timers:
- **Black Hole (BH)**: 60s cooldown, 35s duration
- **Chrono Field (CF)**: 90s cooldown, 34s duration

## Getting Started

### Install Dependencies

```bash
npm install
```

### Development

```bash
npm run dev
```

### Build for Production

```bash
npm run build
```

### Preview Production Build

```bash
npm run preview
```

## Usage

1. **Add Timers**: Click the "+ Add Timer" button to add new timers
2. **Configure Timers**: Edit the name, cooldown, and duration for each timer
3. **Remove Timers**: Click the × button to remove a timer
4. **View Metrics**: Check the Timeline Visualization panel to see:
   - How buffs overlap on the timeline
   - Individual timer statistics (uptime, alone time, overlap effectiveness)
   - Overall overlap statistics

## Technology Stack

- **Svelte 5**: Modern reactive UI framework
- **Vite**: Fast build tool and dev server
- **CSS**: Custom styling with dark/light mode support
