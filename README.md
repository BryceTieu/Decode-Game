# Decode Simulator

Decode Simulator is a browser-based path planning and simulation tool for the Decode game workflow. It provides a visual environment to design robot trajectories, tune control points, simulate playback timing, and export code-ready path data.

This project is built with Svelte + TypeScript and runs entirely in the browser for local development and deployment.

## Table of Contents

- [Overview](#overview)
- [Core Capabilities](#core-capabilities)
- [Tech Stack](#tech-stack)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Available Scripts](#available-scripts)
- [Project Structure](#project-structure)
- [Usage Guide](#usage-guide)
- [Build and Deployment](#build-and-deployment)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [License](#license)

## Overview

Decode Simulator is designed to support iterative autonomous development by combining path authoring, visualization, and export tools in one interface. Teams can draft trajectories, evaluate timing and sequencing, and generate code artifacts for downstream robot logic.

## Core Capabilities

- Interactive field editing with draggable points, control points, and obstacle tools
- Path sequencing and timeline-aware playback controls
- Robot orientation and motion preview with configurable settings
- Multi-path workflow support for coordinated path sets
- Code export options for Java and points-based formats
- Field snapshot export to image for strategy reviews and documentation
- Browser-focused file workflow for save/load and session persistence

## Tech Stack

- Svelte 4
- TypeScript 5
- Vite 5
- Tailwind CSS
- D3 and Two.js for rendering and geometry workflows

## Prerequisites

Install the following before setup:

- Node.js 18.18+ (Node.js 20 LTS recommended)
- npm 9+

Check versions:

```bash
node -v
npm -v
```

## Quick Start

1. Clone the repository.
2. Move into the project directory.
3. Install dependencies.
4. Start the development server.

```bash
git clone <your-repository-url>
cd "Decode Game/Decode Simulator"
npm install
npm run dev
```

After startup, Vite prints a local URL (typically http://localhost:5173).

## Available Scripts

- `npm run dev`: Start the local development server with hot reload
- `npm run build`: Create a production build in the dist output
- `npm run preview`: Serve the production build locally for verification
- `npm run check`: Run Svelte and TypeScript checks

## Project Structure

Key directories:

- `src/`: Application source code
- `src/lib/`: Primary UI modules and feature panels
- `src/lib/components/`: Reusable interface components
- `src/utils/`: Geometry, drawing, history, animation, and export utilities
- `src/config/`: Default values and field/robot configuration
- `public/`: Static assets and service worker files

## Usage Guide

1. Launch the app with `npm run dev`.
2. Create or load a path project.
3. Add and refine waypoints/control points.
4. Configure sequence timing and robot settings.
5. Validate motion with playback and timeline markers.
6. Export trajectories/code or save field snapshots as needed.

## Build and Deployment

Generate an optimized build:

```bash
npm run build
```

Preview the build locally:

```bash
npm run preview
```

Deploy the generated dist assets to your preferred static hosting platform.

## Troubleshooting

- If dependencies fail to install, remove `node_modules` and `package-lock.json`, then run `npm install` again.
- If type checks fail, run `npm run check` and resolve reported Svelte/TypeScript issues.
- If the dev server fails to start, verify Node.js and npm versions meet the prerequisites.

## Contributing

Contributions are welcome. For quality and consistency:

1. Create a feature branch.
2. Keep changes focused and reviewable.
3. Run `npm run check` before opening a pull request.
4. Include a clear summary of functional changes and testing performed.

## License

See the LICENSE file in this repository for licensing terms.
