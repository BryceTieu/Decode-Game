<script lang="ts">
  import type {
    Line,
    BasePoint,
    Settings,
    Point,
    SequenceItem,
    Shape,
  } from "./types";
  import * as d3 from "d3";
  import {
    snapToGrid,
    gridSize,
    currentFilePath,
    isUnsaved,
    showGrid,
    dualPathMode,
    secondFilePath,
    activePaths,
  } from "./stores";
  import Two from "two.js";
  import type { Path } from "two.js/src/path";
  import type { Line as PathLine } from "two.js/src/shapes/line";
  import ControlTab from "./lib/ControlTab.svelte";
  import Navbar from "./lib/Navbar.svelte";
  import MathTools from "./lib/MathTools.svelte";
  import SaveDialog from "./lib/components/SaveDialog.svelte";
  import DualPathSaveDialog from "./lib/components/DualPathSaveDialog.svelte";
  import _ from "lodash";
  import hotkeys from "hotkeys-js";
  import { createAnimationController } from "./utils/animation";
  import { calculatePathTime, getAnimationDuration } from "./utils";

  import {
    calculateRobotState,
    generateGhostPathPoints,
    generateOnionLayers,
  } from "./utils";
  import {
    easeInOutQuad,
    getCurvePoint,
    getRandomColor,
    quadraticToCubic,
    radiansToDegrees,
    shortestRotation,
    getAngularDifference,
    downloadTrajectory,
    loadTrajectoryFromFile,
    loadRobotImage,
    updateRobotImageDisplay,
  } from "./utils";
  import {
    POINT_RADIUS,
    LINE_WIDTH,
    DEFAULT_ROBOT_WIDTH,
    DEFAULT_ROBOT_HEIGHT,
    DEFAULT_SETTINGS,
    FIELD_SIZE,
    getDefaultStartPoint,
    getDefaultLines,
    getDefaultShapes,
  } from "./config";
  import { loadSettings, saveSettings } from "./utils/settingsPersistence";
  import * as browserFileStore from "./utils/browserFileStore";
  import { onMount, tick } from "svelte";
  import { debounce } from "lodash";
  import { createHistory, type AppState } from "./utils/history";
  // Browser-only build: file operations use the browser file store and
  // localStorage. Electron-specific APIs have been removed.

  // Keep existing visualizer logic intact while bootstrapping a drive simulator.
  const simulatorMode = true;

  function normalizeLines(input: Line[]): Line[] {
    return (input || []).map((line) => ({
      ...line,
      id: line.id || `line-${Math.random().toString(36).slice(2)}`,
      controlPoints: line.controlPoints || [],
      color: line.color || getRandomColor(),
      name: line.name || "",
      waitBeforeMs: Math.max(
        0,
        Number(line.waitBeforeMs ?? line.waitBefore?.durationMs ?? 0),
      ),
      waitAfterMs: Math.max(
        0,
        Number(line.waitAfterMs ?? line.waitAfter?.durationMs ?? 0),
      ),
      waitBeforeName: line.waitBeforeName ?? line.waitBefore?.name ?? "",
      waitAfterName: line.waitAfterName ?? line.waitAfter?.name ?? "",
    }));
  }

  // Canvas state
  let two: Two;
  let twoElement: HTMLDivElement;
  let width = 0;
  let height = 0;
  // Robot state
  $: robotWidth = settings?.rWidth || DEFAULT_ROBOT_WIDTH;
  $: robotHeight = settings?.rHeight || DEFAULT_ROBOT_HEIGHT;
  let robotXY: BasePoint = { x: 0, y: 0 };
  let robotHeading: number = 0;
  // Animation state
  let percent: number = 0;
  let playing = false;
  let animationFrame: number;
  let startTime: number | null = null;
  let previousTime: number | null = null;
  // Save dialog state
  let showSaveDialog = false;
  let showDualPathSaveDialog = false;
  let isSaving = false;
  // Path data
  let settings: Settings = { ...DEFAULT_SETTINGS };
  let startPoint: Point = getDefaultStartPoint();
  let lines: Line[] = normalizeLines(getDefaultLines());
  let sequence: SequenceItem[] = lines.map((ln) => ({
    kind: "path",
    lineId: ln.id!,
  }));
  let shapes: Shape[] = getDefaultShapes();
  let optimizingLineIds: Record<string, boolean> = {};
  let optimizingAll = false;

  // Second path data (for alliance coordination) - DEPRECATED, use additionalPaths
  let secondStartPoint: Point | null = null;
  let secondLines: Line[] = [];
  let secondSequence: SequenceItem[] = [];
  let secondShapes: Shape[] = [];

  // Multiple paths data (new system - supports up to 4 paths total)
  interface AdditionalPathData {
    filePath: string;
    startPoint: Point | null;
    lines: Line[];
    sequence: SequenceItem[];
    shapes: Shape[];
    settings: Settings;
    color?: string; // Optional custom color for this path
  }
  let additionalPaths: AdditionalPathData[] = [];

  type SimBall = {
    id: string;
    x: number;
    y: number;
    vx: number;
    vy: number;
    r: number;
    color: "purple" | "green";
    collected: boolean;
    scored: boolean;
    fromRelease?: boolean;
  };

  type SimTransferBall = {
    id: string;
    goalId: string;
    color: "purple" | "green";
    phase: "in-goal" | "to-tube" | "in-tube";
    stored: boolean;
    holdMs: number;
    x: number;
    y: number;
    vx: number;
    vy: number;
    r: number;
    overflowEject?: boolean;
    preOverflow?: boolean;
  };

  type SimGoal = {
    id: string;
    x: number;
    y: number;
    r: number;
    cornerX: number;
    cornerY: number;
    legBackPx: number;
    legSidePx: number;
    openingWidthPx: number;
    openingDepthPx: number;
    label: string;
    side: "left" | "right";
  };

  type SimLever = {
    id: string;
    x: number;
    y: number;
    width: number;
    height: number;
    label: string;
    side: "left" | "right";
  };

  type SimClassifier = {
    id: string;
    x: number;
    y: number;
    width: number;
    height: number;
    side: "left" | "right";
    goalId: string;
    leverId: string;
  };

  type SimProjectile = {
    id: string;
    color: "purple" | "green";
    startX: number;
    startY: number;
    targetX: number;
    targetY: number;
    goalId: string;
    elapsedMs: number;
    durationMs: number;
    arcHeightPx: number;
  };

  type SimPoint = { x: number; y: number };

  type SimRobotState = {
    id: string;
    xy: BasePoint;
    heading: number;
    velX: number;
    velY: number;
    rotVel: number;
    heldBallColors: Array<"purple" | "green">;
    prevButtons: boolean[];
    lastShotMs: number;
    turretAngleRad: number;
  };

  const simShootButtonIndex = 0; // A / Cross
  const simResetButtonIndex = 3; // Y / Triangle
  const simIntakeButtonIndexes = [5, 7]; // RB + RT
  const simAutoControllerSelection = -1;
  const simKeyboardControllerSelection = -2;

  let simConnectedControllers: Array<{ index: number; name: string }> = [];
  let simControllerSummary = "P1: No controller | P2: No controller";
  let simControllerSelections: number[] = [-1, -1];
  let simKeyboardState = {
    moveUp: false,
    moveLeft: false,
    moveDown: false,
    moveRight: false,
    turnLeft: false,
    turnRight: false,
    intake: false,
    shoot: false,
  };
  let simRobots: SimRobotState[] = [];
  let simBalls: SimBall[] = [];
  let simGoals: SimGoal[] = [];
  let simLevers: SimLever[] = [];
  let simClassifiers: SimClassifier[] = [];
  let simGoalInventories: Record<string, Array<"purple" | "green">> = {};
  let simReleaseQueues: Record<string, Array<"purple" | "green">> = {};
  let simNextReleaseAtMs: Record<string, number> = {};
  let simTransferBalls: SimTransferBall[] = [];
  let simProjectiles: SimProjectile[] = [];
  let simScore = 0;
  let simColorScores: Record<"purple" | "green", number> = { purple: 0, green: 0 };
  let simMaxHeldBalls = 3;
  let simInitialized = false;
  let simLastFrameTime = 0;
  let simAnimationFrame = 0;
  let simLastLeverTouchMs = -1000;
  let simClassifierBottomOpen: Record<string, boolean> = {};
  let simLeverOpenUntilMs: Record<string, number> = {};
  let simLeverLastBallContactMs: Record<string, number> = {};
  let simMatchStartMs = 0;
  let simMatchRemainingMs = 120000;

  const simBallRadiusPx = 14;
  const simInventoryBallRadiusPx = 13;
  const simTubeGravity = 420;
  const simTubeBounce = 0.25;
  const simTurretRotationSpeedRadPerSec = Math.PI * 1.5;  // ~270 deg/sec
  let simMoveSpeedPxPerSec = 430;
  let simTurnSpeedDegPerSec = 340;
  let simAccel = 2000;
  let simDecel = 2400;
  let simRotAccel = 3200;
  let simRotDecel = 4500;
  let simShootRepeatMs = 170;
  const simLeverCooldownMs = 500;
  const simLeverOpenDurationMs = 600;
  const simLeverAutoCloseNoContactMs = 500;
  const simShotMinDurationMs = 480;
  const simShotMaxDurationMs = 1200;
  const simReleaseIntervalMs = 120;
  const simBallDampingPerSec = 3.4;
  const simReleasedBallDampingPerSec = simBallDampingPerSec / 5;
  const simReleaseMomentumScale = 1.55;
  const simClassifierWidth = 56;
  const simWallBounce = 0.58;
  const simDriverPeriodMs = 120000;

  $: simMatchActive = simMatchRemainingMs > 0;
  $: simMatchClockText = formatMatchClock(simMatchRemainingMs);

  $: simHeldBallsByRobot = simRobots.map((robot) => robot.heldBallColors.length);
  $: simHeldBalls = simHeldBallsByRobot.reduce((sum, held) => sum + held, 0);

  function createSimRobot(id: string, xPos: number, yPos: number): SimRobotState {
    return {
      id,
      xy: { x: xPos, y: yPos },
      heading: 0,
      velX: 0,
      velY: 0,
      rotVel: 0,
      heldBallColors: [],
      prevButtons: [],
      lastShotMs: -100000,
      turretAngleRad: -Math.PI / 2,
    };
  }

  function createDefaultSimRobots() {
    if (!width || !height) return [];
    return [
      createSimRobot("robot-1", width * 0.42, height * 0.5),
      createSimRobot("robot-2", width * 0.58, height * 0.5),
    ];
  }

  function clampPoseToField(pose: BasePoint) {
    if (!width || !height) return;
    const halfW = x(robotWidth) / 2;
    const halfH = x(robotHeight) / 2;
    pose.x = Math.max(halfW, Math.min(width - halfW, pose.x));
    pose.y = Math.max(halfH, Math.min(height - halfH, pose.y));
  }

  function createSimBalls(count: number): SimBall[] {
    if (!width || !height) return [];
    const margin = 42;
    const balls: SimBall[] = [];
    const purpleCount = Math.floor((count * 2) / 3);
    const greenCount = count - purpleCount;
    const colors: Array<"purple" | "green"> = [
      ...Array.from({ length: purpleCount }, () => "purple" as const),
      ...Array.from({ length: greenCount }, () => "green" as const),
    ];

    for (let i = 0; i < count; i++) {
      balls.push({
        id: `ball-${i + 1}`,
        x: margin + Math.random() * Math.max(1, width - margin * 2),
        y: margin + Math.random() * Math.max(1, height - margin * 2),
        vx: (Math.random() - 0.5) * 24,
        vy: (Math.random() - 0.5) * 24,
        r: simBallRadiusPx,
        color: colors[i],
        collected: false,
        scored: false,
        fromRelease: false,
      });
    }

    return balls;
  }

  function createSimGoals(): SimGoal[] {
    if (!width || !height) return [];
    const pxPerIn = Math.min(width, height) / 144;
    const legPx = Math.max(42, 27 * pxPerIn);
    const openingWidthPx = Math.max(30, 26.5 * pxPerIn);
    const openingDepthPx = Math.max(18, 18.3 * pxPerIn);
    const r = Math.max(18, openingDepthPx * 0.52);
    
    // Goals positioned above inventory, centered on classifiers
    const gapFromWall = 4;
    const classifierWidth = simClassifierWidth;
    const leftGoalX = gapFromWall + classifierWidth / 2;
    const rightGoalX = width - gapFromWall - classifierWidth / 2;
    const goalY = 24;  // Near top, above inventory

    return [
      {
        id: "goal-left",
        x: leftGoalX,
        y: goalY,
        r,
        cornerX: 0,
        cornerY: 0,
        legBackPx: legPx,
        legSidePx: legPx,
        openingWidthPx,
        openingDepthPx,
        label: "Top Left Goal",
        side: "left",
      },
      {
        id: "goal-right",
        x: rightGoalX,
        y: goalY,
        r,
        cornerX: width,
        cornerY: 0,
        legBackPx: legPx,
        legSidePx: legPx,
        openingWidthPx,
        openingDepthPx,
        label: "Top Right Goal",
        side: "right",
      },
    ];
  }

  function createSimLevers(classifierWidth: number): SimLever[] {
    if (!width || !height) return [];
    const leverWidth = classifierWidth + 26;
    const leverHeight = 18;
    return [
      {
        id: "lever-left",
        x: classifierWidth / 2 + 10,
        y: height / 2,
        width: leverWidth,
        height: leverHeight,
        label: "Left Lever",
        side: "left",
      },
      {
        id: "lever-right",
        x: width - (classifierWidth / 2 - 2),
        y: height / 2,
        width: leverWidth,
        height: leverHeight,
        label: "Right Lever",
        side: "right",
      },
    ];
  }

  function createSimClassifiers(goals: SimGoal[], levers: SimLever[]): SimClassifier[] {
    const classifierWidth = simClassifierWidth;
    const gapFromWall = 4;

    const leftGoal = goals.find((goal) => goal.side === "left");
    const rightGoal = goals.find((goal) => goal.side === "right");
    const leftLever = levers.find((lever) => lever.side === "left");
    const rightLever = levers.find((lever) => lever.side === "right");

    const leftY = (leftGoal?.legSidePx ?? 56) + 8;
    const rightY = (rightGoal?.legSidePx ?? 56) + 8;
    const leftHeight = Math.max(52, (leftLever?.y ?? height / 2) - (leftLever?.height ?? 14) / 2 - leftY - 8);
    const rightHeight = Math.max(52, (rightLever?.y ?? height / 2) - (rightLever?.height ?? 14) / 2 - rightY - 8);

    return [
      {
        id: "classifier-left",
        x: gapFromWall,
        y: leftY,
        width: classifierWidth,
        height: leftHeight,
        side: "left",
        goalId: leftGoal?.id || "goal-left",
        leverId: leftLever?.id || "lever-left",
      },
      {
        id: "classifier-right",
        x: width - classifierWidth - gapFromWall,
        y: rightY,
        width: classifierWidth,
        height: rightHeight,
        side: "right",
        goalId: rightGoal?.id || "goal-right",
        leverId: rightLever?.id || "lever-right",
      },
    ];
  }

  function createEmptyGoalInventory(goals: SimGoal[]) {
    return goals.reduce(
      (acc, goal) => {
        acc[goal.id] = [];
        return acc;
      },
      {} as Record<string, Array<"purple" | "green">>,
    );
  }

  function createEmptyReleaseQueue(goals: SimGoal[]) {
    return goals.reduce(
      (acc, goal) => {
        acc[goal.id] = [];
        return acc;
      },
      {} as Record<string, Array<"purple" | "green">>,
    );
  }

  function createEmptyReleaseTimer(goals: SimGoal[]) {
    return goals.reduce(
      (acc, goal) => {
        acc[goal.id] = 0;
        return acc;
      },
      {} as Record<string, number>,
    );
  }

  function randomFieldPoint(margin = 42): SimPoint {
    return {
      x: margin + Math.random() * Math.max(1, width - margin * 2),
      y: margin + Math.random() * Math.max(1, height - margin * 2),
    };
  }

  function getClosestGoal(fromX: number, fromY: number): SimGoal | undefined {
    let closest: SimGoal | undefined;
    let bestDist = Number.POSITIVE_INFINITY;

    for (const goal of simGoals) {
      const dist = Math.hypot(goal.x - fromX, goal.y - fromY);
      if (dist < bestDist) {
        bestDist = dist;
        closest = goal;
      }
    }

    return closest;
  }

  function setControllerSelection(robotIdx: number, rawValue: string) {
    const parsed = Number(rawValue);
    if (Number.isNaN(parsed)) return;
    const next = [...simControllerSelections];
    next[robotIdx] = parsed;
    simControllerSelections = next;
    if (robotIdx === 0 && parsed !== simKeyboardControllerSelection) {
      clearSimKeyboardState();
    }
  }

  function handleControllerSelectionChange(robotIdx: number, event: Event) {
    const select = event.currentTarget as HTMLSelectElement | null;
    if (!select) return;
    setControllerSelection(robotIdx, select.value);
  }

  function clearSimKeyboardState() {
    simKeyboardState = {
      moveUp: false,
      moveLeft: false,
      moveDown: false,
      moveRight: false,
      turnLeft: false,
      turnRight: false,
      intake: false,
      shoot: false,
    };
  }

  function isKeyboardControllerSelected(robotIdx: number) {
    return simControllerSelections[robotIdx] === simKeyboardControllerSelection;
  }

  function isEditableKeyboardTarget(target: EventTarget | null) {
    const element = target instanceof HTMLElement ? target : null;
    if (!element) return false;

    const editableTag = ["INPUT", "TEXTAREA", "SELECT"].includes(element.tagName);
    return editableTag || element.isContentEditable;
  }

  function updateKeyboardControlState(code: string, pressed: boolean) {
    switch (code) {
      case "KeyW":
        simKeyboardState.moveUp = pressed;
        return true;
      case "KeyA":
        simKeyboardState.moveLeft = pressed;
        return true;
      case "KeyS":
        simKeyboardState.moveDown = pressed;
        return true;
      case "KeyD":
        simKeyboardState.moveRight = pressed;
        return true;
      case "KeyJ":
        simKeyboardState.turnLeft = pressed;
        return true;
      case "KeyL":
        simKeyboardState.turnRight = pressed;
        return true;
      case "ShiftLeft":
        simKeyboardState.intake = pressed;
        return true;
      case "Backslash":
        simKeyboardState.shoot = pressed;
        return true;
      default:
        return false;
    }
  }

  function handleSimulatorKeyboard(event: KeyboardEvent, pressed: boolean) {
    if (!simulatorMode || !isKeyboardControllerSelected(0)) return;
    if (isEditableKeyboardTarget(event.target)) return;

    if (updateKeyboardControlState(event.code, pressed)) {
      event.preventDefault();
    }
  }

  function getNormalizedInputVector(xValue: number, yValue: number) {
    const magnitude = Math.hypot(xValue, yValue);
    if (magnitude <= 1) {
      return { x: xValue, y: yValue };
    }

    return {
      x: xValue / magnitude,
      y: yValue / magnitude,
    };
  }

  function getKeyboardMoveVector() {
    const xAxis = Number(simKeyboardState.moveRight) - Number(simKeyboardState.moveLeft);
    const yAxis = Number(simKeyboardState.moveDown) - Number(simKeyboardState.moveUp);
    return getNormalizedInputVector(xAxis, yAxis);
  }

  function getKeyboardTurnInput() {
    return Number(simKeyboardState.turnRight) - Number(simKeyboardState.turnLeft);
  }

  function getConnectedGamepads(): Gamepad[] {
    if (typeof navigator === "undefined" || !navigator.getGamepads) return [];
    return Array.from(navigator.getGamepads())
      .filter((pad): pad is Gamepad => !!pad && pad.connected)
      .sort((a, b) => a.index - b.index);
  }

  function resolveAssignedGamepads(gamepads: Gamepad[]) {
    const assigned: Array<Gamepad | null> = [null, null];
    const used = new Set<number>();

    for (let idx = 0; idx < 2; idx++) {
      const selectedIdx = simControllerSelections[idx] ?? -1;
      let chosen: Gamepad | null = null;

      if (selectedIdx >= 0) {
        chosen = gamepads.find((pad) => pad.index === selectedIdx) ?? null;
        if (chosen && used.has(chosen.index)) {
          chosen = null;
        }
      } else if (selectedIdx === simAutoControllerSelection) {
        chosen = gamepads.find((pad) => !used.has(pad.index)) ?? null;
      }

      if (chosen) {
        used.add(chosen.index);
      }
      assigned[idx] = chosen;
    }

    return assigned;
  }

  function getClassifierForGoal(goalId: string) {
    return simClassifiers.find((classifier) => classifier.goalId === goalId);
  }

  function isLeverOpen(lever: SimLever) {
    const classifier = simClassifiers.find((c) => c.leverId === lever.id);
    if (!classifier) return false;
    return !!simClassifierBottomOpen[classifier.goalId];
  }

  function formatMatchClock(ms: number) {
    const clamped = Math.max(0, ms);
    const totalSeconds = Math.ceil(clamped / 1000);
    const minutes = Math.floor(totalSeconds / 60);
    const seconds = totalSeconds % 60;
    return `${String(minutes).padStart(2, "0")}:${String(seconds).padStart(2, "0")}`;
  }

  function updateMatchClock(nowMs: number) {
    if (simMatchStartMs <= 0) {
      simMatchStartMs = nowMs;
      simMatchRemainingMs = simDriverPeriodMs;
      return;
    }

    const elapsed = Math.max(0, nowMs - simMatchStartMs);
    simMatchRemainingMs = Math.max(0, simDriverPeriodMs - elapsed);
  }

  function getClassifierInventorySlots(classifier: SimClassifier) {
    const colors = simGoalInventories[classifier.goalId] || [];
    const shown = colors.slice(0, 9);
    const count = shown.length;
    const topPadding = simInventoryBallRadiusPx + 4;
    const bottomPadding = simInventoryBallRadiusPx + 4;
    const available = Math.max(0, classifier.height - topPadding - bottomPadding);
    const spacing =
      count <= 1
        ? 0
        : Math.min(simInventoryBallRadiusPx * 2 + 2, available / Math.max(1, count - 1));
    const centerX = classifier.x + classifier.width / 2;
    const baseY = classifier.y + classifier.height - bottomPadding;

    return shown.map((color, idx) => {
      return {
        color,
        r: simInventoryBallRadiusPx,
        x: centerX,
        y: baseY - idx * spacing,
      };
    });
  }

  function getClassifierInventoryTarget(goalId: string) {
    const classifier = getClassifierForGoal(goalId);
    if (!classifier) return null;

    const count = Math.min(9, (simGoalInventories[goalId] || []).length + 1);
    const slotIndex = Math.min(8, (simGoalInventories[goalId] || []).length);
    const topPadding = simInventoryBallRadiusPx + 4;
    const bottomPadding = simInventoryBallRadiusPx + 4;
    const available = Math.max(0, classifier.height - topPadding - bottomPadding);
    const spacing =
      count <= 1
        ? 0
        : Math.min(simInventoryBallRadiusPx * 2 + 2, available / Math.max(1, count - 1));
    const centerX = classifier.x + classifier.width / 2;
    const baseY = classifier.y + classifier.height - bottomPadding;

    return {
      x: centerX,
      y: baseY - slotIndex * spacing,
    };
  }

  function updateTurretAimForRobot(robot: SimRobotState) {
    const targetGoal = getClosestGoal(robot.xy.x, robot.xy.y);
    if (!targetGoal) return;
    const targetAngle = Math.atan2(targetGoal.y - robot.xy.y, targetGoal.x - robot.xy.x);
    // Calculate the shortest angular difference in radians
    const currentDeg = robot.turretAngleRad * (180 / Math.PI);
    const targetDeg = targetAngle * (180 / Math.PI);
    const diff = getAngularDifference(currentDeg, targetDeg);
    const maxRotation = simTurretRotationSpeedRadPerSec / 60;  // Per frame at ~60fps
    const clampedDiff = Math.max(-maxRotation, Math.min(maxRotation, diff * Math.PI / 180));
    robot.turretAngleRad += clampedDiff;
  }

  function refreshControllerStatus(assignments?: Array<Gamepad | null>) {
    const gamepads = getConnectedGamepads();
    simConnectedControllers = gamepads.map((pad) => ({ index: pad.index, name: pad.id }));

    const active = assignments ?? resolveAssignedGamepads(gamepads);
    const p1 = isKeyboardControllerSelected(0)
      ? "Keyboard"
      : active[0]
        ? active[0].id
        : "No controller";
    const p2 = active[1] ? active[1].id : "No controller";
    simControllerSummary = `P1: ${p1} | P2: ${p2}`;
  }

  function spawnGoalToInventoryTransfer(
    goalId: string,
    color: "purple" | "green",
    preOverflow = false,
  ) {
    const goal = simGoals.find((g) => g.id === goalId);
    if (!goal) return;

    simTransferBalls = [
      ...simTransferBalls,
      {
        id: `transfer-${goalId}-${Date.now()}-${Math.random().toString(36).slice(2, 6)}`,
        goalId,
        color,
        phase: "in-goal",
        stored: false,
        holdMs: 0,
        x: goal.x,
        y: goal.y,
        vx: 0,
        vy: 0,
        r: simBallRadiusPx,
        preOverflow,
      },
    ];
  }

  function getPendingInboundTransferCount(goalId: string) {
    return simTransferBalls.filter(
      (ball) =>
        ball.goalId === goalId &&
        !ball.stored &&
        !ball.overflowEject,
    ).length;
  }

  function addBallToInventory(goalId: string, color: "purple" | "green") {
    const current = simGoalInventories[goalId] || [];
    if (current.length >= 9) {
      // Inventory is full, reject this ball
      return false;
    }

    simGoalInventories = {
      ...simGoalInventories,
      [goalId]: [...current, color],
    };

    return true;
  }

  function updateTransferBalls(dtMs: number, nowMs: number) {
    const dt = dtMs / 1000;
    const exitedBalls: Array<{ goalId: string; color: "purple" | "green"; x: number; y: number; vy: number }> = [];
    const contactGoals = new Set<string>();

    // Phase transitions: in-goal and to-tube
    for (const ball of simTransferBalls) {
      const goal = simGoals.find((g) => g.id === ball.goalId);
      const classifier = getClassifierForGoal(ball.goalId);
      if (!classifier || !goal) continue;

      if (ball.phase === "in-goal") {
        ball.holdMs += dtMs;
        ball.x = goal.x;
        ball.y = goal.y + goal.openingDepthPx * 0.28;
        if (ball.holdMs >= 160) {
          ball.phase = "to-tube";
          ball.x = goal.x;
          ball.y = goal.y + goal.openingDepthPx * 0.86;
          ball.vx = 0;
          ball.vy = 110;
          ball.holdMs = 0;  // Reset timer for to-tube phase
        }
      } else if (ball.phase === "to-tube") {
        ball.holdMs += dtMs;  // Accumulate time in to-tube phase for inking delay
        const targetX = classifier.x + classifier.width / 2;
        const targetY = classifier.y + ball.r + 4;
        const dx = targetX - ball.x;
        const dy = targetY - ball.y;
        const dist = Math.hypot(dx, dy);

        ball.vx += dx * dt * 2.4;
        ball.vy += dy * dt * 2.6;
        ball.vx *= 0.988;
        ball.vy *= 0.988;
        ball.x += ball.vx * dt;
        ball.y += ball.vy * dt;

        // Only transition to in-tube after 50ms (small inking delay) AND ball is close enough
        if (dist < Math.max(8, ball.r * 0.72) && ball.holdMs >= 50) {
          ball.phase = "in-tube";
          ball.r = simInventoryBallRadiusPx;
          ball.vx = 0;
          ball.vy = 20;
          if (!ball.stored) {
            const stored = ball.preOverflow ? false : addBallToInventory(ball.goalId, ball.color);
            ball.stored = stored;
            if (!stored) {
              // Overflow ball rides on top and spills back onto the field.
              ball.overflowEject = true;
              ball.vy = -220;
            }
          }
        }
      }
    }

    // Remove dead transfer balls (missing goal/classifier)
    simTransferBalls = simTransferBalls.filter((b) => {
      const goal = simGoals.find((g) => g.id === b.goalId);
      const classifier = getClassifierForGoal(b.goalId);
      return !!(goal && classifier);
    });

    // Tube physics per classifier
    for (const classifier of simClassifiers) {
      const goalId = classifier.goalId;
      const tubeBalls = simTransferBalls.filter(
        (b) => b.goalId === goalId && b.phase === "in-tube",
      );
      if (tubeBalls.length === 0) {
        continue;
      }

      const centerX = classifier.x + classifier.width / 2;
      const isOpen = simClassifierBottomOpen[goalId] || false;
      const r = simInventoryBallRadiusPx;
      const wallPad = 4;
      const bottomWallY = classifier.y + classifier.height - r - wallPad;
      const topWallY = classifier.y + r + wallPad;

      // Gravity + move
      for (const ball of tubeBalls) {
        if (ball.overflowEject) {
          const exitX =
            classifier.side === "left"
              ? classifier.x + classifier.width - ball.r - 2
              : classifier.x + ball.r + 2;
          const dxExit = exitX - ball.x;

          ball.vx += dxExit * dt * 20;
          ball.vx *= Math.pow(0.86, dt * 60);
          ball.x += ball.vx * dt;

          // Ride over the top stack first, then plop out near the lever-end.
          if (Math.abs(dxExit) > 3) {
            const rideY = topWallY + ball.r * 0.45;
            ball.y += (rideY - ball.y) * Math.min(1, dt * 10);
            ball.vy = Math.min(ball.vy, 20);
          } else {
            ball.vy += simTubeGravity * dt * 1.15;
            ball.y += ball.vy * dt;
            ball.vy *= Math.pow(0.975, dt * 60);
          }
          continue;
        }

        ball.vy += simTubeGravity * dt;
        ball.x = centerX;
        ball.y += ball.vy * dt;
        ball.vy *= Math.pow(0.97, dt * 60);
      }

      // Sort by Y descending (bottom-most first)
      tubeBalls.sort((a, b) => b.y - a.y);

      // Bottom wall (only when closed)
      if (!isOpen) {
        for (const ball of tubeBalls) {
          if (ball.overflowEject) continue;
          if (ball.y > bottomWallY) {
            ball.y = bottomWallY;
            ball.vy = -Math.abs(ball.vy) * simTubeBounce;
            if (Math.abs(ball.vy) < 3) ball.vy = 0;
          }
        }
      }

      // Ball-ball 1D vertical collision (bottom to top)
      for (let i = 0; i < tubeBalls.length; i++) {
        for (let j = i + 1; j < tubeBalls.length; j++) {
          const lower = tubeBalls[i];
          const upper = tubeBalls[j];
          if (lower.overflowEject || upper.overflowEject) continue;
          const minDist = lower.r + upper.r;
          const gap = lower.y - upper.y;
          if (gap > 0 && gap < minDist) {
            const overlap = minDist - gap;
            lower.y += overlap * 0.5;
            upper.y -= overlap * 0.5;
            const relVel = lower.vy - upper.vy;
            if (relVel < 0) {
              const impulse = relVel * 0.5 * (1 + simTubeBounce);
              lower.vy -= impulse;
              upper.vy += impulse;
            }
          }
        }
      }

      // Re-apply bottom wall after collision resolution
      if (!isOpen) {
        for (const ball of tubeBalls) {
          if (ball.overflowEject) continue;
          if (ball.y > bottomWallY) {
            ball.y = bottomWallY;
            if (ball.vy > 0) ball.vy = 0;
          }
        }
      }

      // Top wall
      for (const ball of tubeBalls) {
        if (ball.y < topWallY) {
          ball.y = topWallY;
          if (ball.vy < 0) ball.vy = Math.abs(ball.vy) * simTubeBounce;
        }
      }

    }

    // Balls exiting the lever-end (open gate or overflow ball) → convert to SimBalls
    simTransferBalls = simTransferBalls.filter((ball) => {
      if (ball.phase !== "in-tube") return true;
      const classifier = getClassifierForGoal(ball.goalId);
      if (!classifier) return false;

      const isOpen = simClassifierBottomOpen[ball.goalId] || false;
      if ((isOpen || ball.overflowEject) && ball.y > classifier.y + classifier.height + ball.r) {
        exitedBalls.push({
          goalId: ball.goalId,
          color: ball.color,
          x: ball.x,
          y: ball.y,
          vy: ball.vy,
        });
        contactGoals.add(ball.goalId);
        return false;
      }
      return true;
    });

    // Spawn exited balls as free SimBalls
    for (const exited of exitedBalls) {
      const released: SimBall = {
        id: `released-${exited.goalId}-${Date.now()}-${Math.random().toString(36).slice(2, 6)}`,
        x: exited.x,
        y: exited.y,
        vx:
          (getClassifierForGoal(exited.goalId)?.side === "left" ? -1 : 1) *
          (35 + Math.random() * 25),
        vy: Math.max(50, Math.abs(exited.vy) * 0.6) * simReleaseMomentumScale,
        r: simBallRadiusPx,
        color: exited.color,
        collected: false,
        scored: false,
        fromRelease: true,
      };
      simBalls = [...simBalls, released];

      const currentInventory = [...(simGoalInventories[exited.goalId] || [])];
      const dropIndex = currentInventory.indexOf(exited.color);
      if (dropIndex >= 0) {
        currentInventory.splice(dropIndex, 1);
        simGoalInventories = { ...simGoalInventories, [exited.goalId]: currentInventory };
      }
    }

    if (contactGoals.size > 0) {
      const nextContact = { ...simLeverLastBallContactMs };
      contactGoals.forEach((goalId) => {
        nextContact[goalId] = nowMs;
      });
      simLeverLastBallContactMs = nextContact;
    }
  }

  function spawnReleasedBall(goalId: string, color: "purple" | "green") {
    const classifier = getClassifierForGoal(goalId);
    if (!classifier) return;

    const released: SimBall = {
      id: `released-${goalId}-${Date.now()}-${Math.random().toString(36).slice(2, 6)}`,
      x: classifier.x + classifier.width / 2,
      y: classifier.y + classifier.height + simBallRadiusPx + 2,
      vx: (Math.random() - 0.5) * 8,
      vy: (130 + Math.random() * 90) * simReleaseMomentumScale,
      r: simBallRadiusPx,
      color,
      collected: false,
      scored: false,
      fromRelease: true,
    };

    simBalls = [...simBalls, released];
  }

  function releaseGoalInventoryFromLever(goalId: string, nowMs: number) {
    // Open gate and keep it open for a minimum hold duration.
    simClassifierBottomOpen = { ...simClassifierBottomOpen, [goalId]: true };
    simLeverOpenUntilMs = {
      ...simLeverOpenUntilMs,
      [goalId]: Math.max(simLeverOpenUntilMs[goalId] ?? 0, nowMs + simLeverOpenDurationMs),
    };
    simLeverLastBallContactMs = {
      ...simLeverLastBallContactMs,
      [goalId]: nowMs,
    };
  }

  function updateLeverGateState(nowMs: number) {
    const nextOpen = { ...simClassifierBottomOpen };

    for (const goalId of Object.keys(nextOpen)) {
      if (!nextOpen[goalId]) continue;
      const holdUntil = simLeverOpenUntilMs[goalId] ?? 0;
      const lastContact = simLeverLastBallContactMs[goalId] ?? 0;
      if (nowMs >= holdUntil && nowMs - lastContact >= simLeverAutoCloseNoContactMs) {
        nextOpen[goalId] = false;
      }
    }

    simClassifierBottomOpen = nextOpen;
  }

  function releaseInventoriesForSide(side: "left" | "right", nowMs: number) {
    const matching = simClassifiers.filter((classifier) => classifier.side === side);
    matching.forEach((classifier) => {
      releaseGoalInventoryFromLever(classifier.goalId, nowMs);
    });
  }

  function updateReleaseQueues(nowMs: number) {
    for (const goalId of Object.keys(simReleaseQueues)) {
      const queue = simReleaseQueues[goalId] || [];
      if (!queue.length) continue;
      const nextAt = simNextReleaseAtMs[goalId] ?? 0;
      if (nowMs < nextAt) continue;

      const [color, ...rest] = queue;
      spawnReleasedBall(goalId, color);

      const currentInventory = [...(simGoalInventories[goalId] || [])];
      const dropIndex = currentInventory.indexOf(color);
      if (dropIndex >= 0) {
        currentInventory.splice(dropIndex, 1);
        simGoalInventories = {
          ...simGoalInventories,
          [goalId]: currentInventory,
        };
      }

      simReleaseQueues = {
        ...simReleaseQueues,
        [goalId]: rest,
      };
      simNextReleaseAtMs = {
        ...simNextReleaseAtMs,
        [goalId]: nowMs + simReleaseIntervalMs,
      };
    }
  }

  function scoreBallIntoGoal(goalId: string, color: "purple" | "green", _nowMs: number) {
    const currentInventory = simGoalInventories[goalId] || [];
    const pendingInbound = getPendingInboundTransferCount(goalId);
    const preOverflow = currentInventory.length + pendingInbound >= 9;
    spawnGoalToInventoryTransfer(goalId, color, preOverflow);
    simScore += 1;
    simColorScores = {
      ...simColorScores,
      [color]: (simColorScores[color] || 0) + 1,
    };
  }

  function projectileEase(progress: number) {
    const t = Math.max(0, Math.min(1, progress));
    // Fast launch, then decelerate through the arc.
    return 1 - Math.pow(1 - t, 5);
  }

  function getProjectileRenderState(projectile: SimProjectile) {
    const t = Math.max(0, Math.min(1, projectile.elapsedMs / projectile.durationMs));
    const eased = projectileEase(t);
    const xPos = projectile.startX + (projectile.targetX - projectile.startX) * eased;
    const yLinear = projectile.startY + (projectile.targetY - projectile.startY) * eased;
    const arc = Math.sin(Math.PI * t) * projectile.arcHeightPx;
    return {
      x: xPos,
      y: yLinear - arc,
    };
  }

  function updateProjectiles(dtMs: number, nowMs: number) {
    const completed: Array<{ goalId: string; color: "purple" | "green" }> = [];

    simProjectiles = simProjectiles.filter((projectile) => {
      const nextElapsed = projectile.elapsedMs + dtMs;
      if (nextElapsed >= projectile.durationMs) {
        completed.push({ goalId: projectile.goalId, color: projectile.color });
        return false;
      }
      projectile.elapsedMs = nextElapsed;
      return true;
    });

    completed.forEach(({ goalId, color }) => {
      scoreBallIntoGoal(goalId, color, nowMs);
    });
  }

  function leverSideHitByRobot(lever: SimLever, robotPos: BasePoint) {
    const robotRadius = Math.max(x(robotWidth), x(robotHeight)) / 2;
    const sideBandWidth = 10;
    const edgeX = lever.side === "left" ? lever.x + lever.width / 2 : lever.x - lever.width / 2;
    const minX = edgeX - sideBandWidth / 2;
    const maxX = edgeX + sideBandWidth / 2;
    const minY = lever.y - lever.height / 2;
    const maxY = lever.y + lever.height / 2;
    const nearestX = Math.max(minX, Math.min(robotPos.x, maxX));
    const nearestY = Math.max(minY, Math.min(robotPos.y, maxY));

    // Require the robot center to be on the field-facing side of the lever.
    if (lever.side === "left" && robotPos.x < edgeX) return false;
    if (lever.side === "right" && robotPos.x > edgeX) return false;

    return Math.hypot(robotPos.x - nearestX, robotPos.y - nearestY) <= robotRadius;
  }

  function checkLeverDump(nowMs: number) {
    for (const lever of simLevers) {
      const hit = simRobots.some((robot) => leverSideHitByRobot(lever, robot.xy));
      if (hit) {
        if (nowMs - simLastLeverTouchMs >= simLeverCooldownMs) {
          simLastLeverTouchMs = nowMs;
          releaseInventoriesForSide(lever.side, nowMs);
        }
        return;
      }
    }
  }

  function clampRobotAgainstClassifiers(pose: BasePoint, prevX: number, prevY: number) {
    const radius = Math.max(x(robotWidth), x(robotHeight)) / 2;

    for (const classifier of simClassifiers) {
      const minX = classifier.x - radius;
      const maxX = classifier.x + classifier.width + radius;
      const minY = classifier.y - radius;
      const maxY = classifier.y + classifier.height + radius;

      if (
        pose.x >= minX &&
        pose.x <= maxX &&
        pose.y >= minY &&
        pose.y <= maxY
      ) {
        const leftPen = Math.abs(pose.x - minX);
        const rightPen = Math.abs(maxX - pose.x);
        const topPen = Math.abs(pose.y - minY);
        const bottomPen = Math.abs(maxY - pose.y);
        const minPen = Math.min(leftPen, rightPen, topPen, bottomPen);

        if (minPen === leftPen) {
          pose.x = minX;
        } else if (minPen === rightPen) {
          pose.x = maxX;
        } else if (minPen === topPen) {
          pose.y = minY;
        } else {
          pose.y = maxY;
        }

        // If we are still trapped due to edge cases, snap back to previous safe pose.
        if (
          pose.x >= minX &&
          pose.x <= maxX &&
          pose.y >= minY &&
          pose.y <= maxY
        ) {
          pose.x = prevX;
          pose.y = prevY;
        }
      }
    }
  }

  function updateRobotRobotCollision() {
    if (simRobots.length < 2) return;
    const a = simRobots[0];
    const b = simRobots[1];
    const robotRadius = Math.max(x(robotWidth), x(robotHeight)) / 2;
    const minDist = robotRadius * 2;
    const dx = b.xy.x - a.xy.x;
    const dy = b.xy.y - a.xy.y;
    const dist = Math.hypot(dx, dy);
    if (dist <= 0 || dist >= minDist) return;

    const nx = dx / dist;
    const ny = dy / dist;
    const overlap = minDist - dist;
    a.xy.x -= nx * overlap * 0.5;
    a.xy.y -= ny * overlap * 0.5;
    b.xy.x += nx * overlap * 0.5;
    b.xy.y += ny * overlap * 0.5;

    const relVel = (b.velX - a.velX) * nx + (b.velY - a.velY) * ny;
    if (relVel < 0) {
      const restitution = 0.25;
      const impulse = (-(1 + restitution) * relVel) / 2;
      a.velX -= impulse * nx;
      a.velY -= impulse * ny;
      b.velX += impulse * nx;
      b.velY += impulse * ny;
    }

    clampPoseToField(a.xy);
    clampPoseToField(b.xy);
  }

  function updateBalls(dtMs: number) {
    const dt = dtMs / 1000;
    const robotRadius = Math.max(x(robotWidth), x(robotHeight)) / 2;

    const nextBalls = simBalls.map((ball) => ({ ...ball }));

    for (const next of nextBalls) {
      if (next.collected || next.scored) continue;

      next.x += next.vx * dt;
      next.y += next.vy * dt;

      const dampingPerSec = next.fromRelease
        ? simReleasedBallDampingPerSec
        : simBallDampingPerSec;
      const damping = Math.max(0, 1 - dampingPerSec * dt);
      next.vx *= damping;
      next.vy *= damping;

      if (next.x - next.r < 0) {
        next.x = next.r;
        next.vx = Math.abs(next.vx) * simWallBounce;
      }
      if (next.x + next.r > width) {
        next.x = width - next.r;
        next.vx = -Math.abs(next.vx) * simWallBounce;
      }
      if (next.y - next.r < 0) {
        next.y = next.r;
        next.vy = Math.abs(next.vy) * simWallBounce;
      }
      if (next.y + next.r > height) {
        next.y = height - next.r;
        next.vy = -Math.abs(next.vy) * simWallBounce;
      }

      // Treat classifiers as solid walls for free balls.
      for (const classifier of simClassifiers) {
        const minX = classifier.x - next.r;
        const maxX = classifier.x + classifier.width + next.r;
        const minY = classifier.y - next.r;
        const maxY = classifier.y + classifier.height + next.r;

        if (next.x >= minX && next.x <= maxX && next.y >= minY && next.y <= maxY) {
          const leftPen = Math.abs(next.x - minX);
          const rightPen = Math.abs(maxX - next.x);
          const topPen = Math.abs(next.y - minY);
          const bottomPen = Math.abs(maxY - next.y);
          const minPen = Math.min(leftPen, rightPen, topPen, bottomPen);

          if (minPen === leftPen) {
            next.x = minX;
            next.vx = -Math.abs(next.vx) * simWallBounce;
          } else if (minPen === rightPen) {
            next.x = maxX;
            next.vx = Math.abs(next.vx) * simWallBounce;
          } else if (minPen === topPen) {
            next.y = minY;
            next.vy = -Math.abs(next.vy) * simWallBounce;
          } else {
            next.y = maxY;
            next.vy = Math.abs(next.vy) * simWallBounce;
          }
        }
      }

      for (const robot of simRobots) {
        const dx = next.x - robot.xy.x;
        const dy = next.y - robot.xy.y;
        const dist = Math.hypot(dx, dy);
        const minDist = robotRadius + next.r;

        if (dist > 0 && dist < minDist) {
          const nx = dx / dist;
          const ny = dy / dist;
          const push = minDist - dist;
          next.x += nx * push;
          next.y += ny * push;
          next.vx += robot.velX * 0.35 + nx * 18;
          next.vy += robot.velY * 0.35 + ny * 18;
        }
      }
    }

    // Pairwise ball collisions for active field balls.
    for (let i = 0; i < nextBalls.length; i++) {
      const a = nextBalls[i];
      if (a.collected || a.scored) continue;

      for (let j = i + 1; j < nextBalls.length; j++) {
        const b = nextBalls[j];
        if (b.collected || b.scored) continue;

        const dx = b.x - a.x;
        const dy = b.y - a.y;
        const dist = Math.hypot(dx, dy);
        const minDist = a.r + b.r;

        if (dist <= 0 || dist >= minDist) continue;

        const nx = dx / dist;
        const ny = dy / dist;
        const overlap = minDist - dist;

        a.x -= nx * overlap * 0.5;
        a.y -= ny * overlap * 0.5;
        b.x += nx * overlap * 0.5;
        b.y += ny * overlap * 0.5;

        const relVel = (b.vx - a.vx) * nx + (b.vy - a.vy) * ny;
        if (relVel < 0) {
          const restitution = 0.72;
          const impulse = (-(1 + restitution) * relVel) / 2;
          a.vx -= impulse * nx;
          a.vy -= impulse * ny;
          b.vx += impulse * nx;
          b.vy += impulse * ny;
        }
      }
    }

    simBalls = nextBalls;
  }

  function getIntakeGeometry(robot: SimRobotState) {
    const headingRad = (robot.heading * Math.PI) / 180;
    const forwardX = Math.cos(headingRad);
    const forwardY = Math.sin(headingRad);
    const rightX = Math.cos(headingRad + Math.PI / 2);
    const rightY = Math.sin(headingRad + Math.PI / 2);
    const backOffset = -x(robotHeight) / 2;
    const halfWidth = x(robotWidth) / 2;
    const centerX = robot.xy.x + forwardX * backOffset;
    const centerY = robot.xy.y + forwardY * backOffset;

    return {
      centerX,
      centerY,
      forwardX,
      forwardY,
      rightX,
      rightY,
      halfWidth,
    };
  }

  function tryIntakeBall(robot: SimRobotState) {
    if (robot.heldBallColors.length >= simMaxHeldBalls) return;
    const intake = getIntakeGeometry(robot);
    let pickedColor: "purple" | "green" | null = null;

    simBalls = simBalls.map((ball) => {
      if (pickedColor || ball.collected || ball.scored) return ball;

      const vx = ball.x - intake.centerX;
      const vy = ball.y - intake.centerY;
      const lateral = vx * intake.rightX + vy * intake.rightY;
      const forward = vx * intake.forwardX + vy * intake.forwardY;

      const touchingIntakeSide =
        Math.abs(lateral) <= intake.halfWidth + ball.r && Math.abs(forward) <= ball.r + 2;

      if (touchingIntakeSide) {
        pickedColor = ball.color;
        return {
          ...ball,
          collected: true,
          vx: 0,
          vy: 0,
        };
      }

      return ball;
    });

    if (pickedColor) {
      robot.heldBallColors = [...robot.heldBallColors, pickedColor];
    }
  }

  function tryShootBall(robot: SimRobotState, nowMs: number) {
    if (robot.heldBallColors.length <= 0 || simGoals.length === 0) return;

    const targetGoal = getClosestGoal(robot.xy.x, robot.xy.y);
    if (!targetGoal) return;

    const nextHeld = [...robot.heldBallColors];
    const shotColor = nextHeld.shift();
    robot.heldBallColors = nextHeld;
    if (!shotColor) return;

    const travelDistance = Math.hypot(targetGoal.x - robot.xy.x, targetGoal.y - robot.xy.y);
    const duration = Math.max(
      simShotMinDurationMs,
      Math.min(simShotMaxDurationMs, travelDistance * 2.9),
    );

    simProjectiles = [
      ...simProjectiles,
      {
        id: `projectile-${Date.now()}-${Math.random().toString(36).slice(2, 6)}`,
        color: shotColor,
        startX: robot.xy.x,
        startY: robot.xy.y,
        targetX: targetGoal.x,
        targetY: targetGoal.y,
        goalId: targetGoal.id,
        elapsedMs: 0,
        durationMs: duration,
        arcHeightPx: Math.max(18, Math.min(64, travelDistance * 0.22)),
      },
    ];

    robot.lastShotMs = nowMs;
  }

  function ensureSimulatorSpatialLayout() {
    if (!simulatorMode || width <= 0 || height <= 0) return;
    if (simGoals.length === 0) {
      simGoals = createSimGoals();
    }

    if (simClassifiers.length === 0 || simLevers.length === 0) {
      const classifierWidth = simClassifierWidth;
      simLevers = createSimLevers(classifierWidth);
      simClassifiers = createSimClassifiers(simGoals, simLevers);
    }

    if (Object.keys(simGoalInventories).length === 0 && simGoals.length > 0) {
      simGoalInventories = createEmptyGoalInventory(simGoals);
    }
  }

  function handleFieldResizeForSimulator() {
    if (!simulatorMode || !simInitialized || width <= 0 || height <= 0) return;

    simGoals = createSimGoals();
    const classifierWidth = simClassifierWidth;
    simLevers = createSimLevers(classifierWidth);
    simClassifiers = createSimClassifiers(simGoals, simLevers);
    simGoalInventories = {
      ...createEmptyGoalInventory(simGoals),
      ...simGoalInventories,
    };
    simReleaseQueues = {
      ...createEmptyReleaseQueue(simGoals),
      ...simReleaseQueues,
    };
    simNextReleaseAtMs = {
      ...createEmptyReleaseTimer(simGoals),
      ...simNextReleaseAtMs,
    };
    simRobots.forEach((robot) => updateTurretAimForRobot(robot));
  }

  function resetSimulatorRobot() {
    if (!width || !height) return;
    simRobots = createDefaultSimRobots();
    if (simRobots[0]) {
      robotXY = { ...simRobots[0].xy };
      robotHeading = simRobots[0].heading;
    }
  }

  function resetSimulatorMatch() {
    if (!width || !height) return;
    simBalls = createSimBalls(24);
    simGoals = createSimGoals();
    const classifierWidth = simClassifierWidth;
    simLevers = createSimLevers(classifierWidth);
    simClassifiers = createSimClassifiers(simGoals, simLevers);
    simGoalInventories = createEmptyGoalInventory(simGoals);
    simReleaseQueues = createEmptyReleaseQueue(simGoals);
    simNextReleaseAtMs = createEmptyReleaseTimer(simGoals);
    simProjectiles = [];
    simTransferBalls = [];
    simRobots = createDefaultSimRobots();
    simClassifierBottomOpen = {};
    simLeverOpenUntilMs = {};
    simLeverLastBallContactMs = {};
    simScore = 0;
    simColorScores = { purple: 0, green: 0 };
    simLastLeverTouchMs = -1000;
    simMatchStartMs = performance.now();
    simMatchRemainingMs = simDriverPeriodMs;
    simRobots.forEach((robot) => updateTurretAimForRobot(robot));
    if (simRobots[0]) {
      robotXY = { ...simRobots[0].xy };
      robotHeading = simRobots[0].heading;
    }
  }

  function applyDeadzone(value: number, deadzone = 0.15) {
    if (Math.abs(value) < deadzone) return 0;
    return value;
  }

  function applyAccelDecel(current: number, target: number, rate: number, dtSec: number): number {
    const diff = target - current;
    const maxStep = rate * dtSec;
    if (Math.abs(diff) <= maxStep) return target;
    return current + Math.sign(diff) * maxStep;
  }

  function safeTuning(value: number, fallback: number, min: number, max: number): number {
    if (!Number.isFinite(value)) return fallback;
    return Math.max(min, Math.min(max, value));
  }

  function updateSimulator(dtMs: number) {
    const dtSec = dtMs / 1000;
    if (dtSec <= 0) return;

    ensureSimulatorSpatialLayout();
    const nowMs = performance.now();
    updateMatchClock(nowMs);
    updateReleaseQueues(nowMs);
    updateProjectiles(dtMs, nowMs);
    updateTransferBalls(dtMs, nowMs);
    updateLeverGateState(nowMs);
    if (simRobots.length < 2) {
      simRobots = createDefaultSimRobots();
    }

    const gamepads = getConnectedGamepads();
    const assignedGamepads = resolveAssignedGamepads(gamepads);
    refreshControllerStatus(assignedGamepads);

    const moveSpeed = safeTuning(simMoveSpeedPxPerSec, 430, 20, 900);
    const turnSpeed = safeTuning(simTurnSpeedDegPerSec, 340, 20, 720);
    const accel = safeTuning(simAccel, 2000, 50, 5000);
    const decel = safeTuning(simDecel, 2400, 50, 5000);
    const turnAccel = safeTuning(simRotAccel, 3200, 50, 5000);
    const turnDecel = safeTuning(simRotDecel, 4500, 50, 5000);
    const shootRepeatMs = safeTuning(simShootRepeatMs, 170, 60, 1000);

    let resetPressed = false;

    for (let robotIdx = 0; robotIdx < simRobots.length; robotIdx++) {
      const robot = simRobots[robotIdx];
      const gamepad = assignedGamepads[robotIdx];
      const useKeyboard = robotIdx === 0 && isKeyboardControllerSelected(robotIdx);
      const prevRobotX = robot.xy.x;
      const prevRobotY = robot.xy.y;

      if (!gamepad && !useKeyboard) {
        robot.velX = applyAccelDecel(robot.velX, 0, decel, dtSec);
        robot.velY = applyAccelDecel(robot.velY, 0, decel, dtSec);
        robot.rotVel = applyAccelDecel(robot.rotVel, 0, turnDecel, dtSec);
        robot.prevButtons = [];
      } else if (useKeyboard) {
        const moveInput = getKeyboardMoveVector();
        const turnInput = getKeyboardTurnInput();
        const targetVelX = moveInput.x * moveSpeed;
        const targetVelY = moveInput.y * moveSpeed;
        const moveRate = moveInput.x !== 0 || moveInput.y !== 0 ? accel : decel;
        robot.velX = applyAccelDecel(robot.velX, targetVelX, moveRate, dtSec);
        robot.velY = applyAccelDecel(robot.velY, targetVelY, moveRate, dtSec);

        if (turnInput !== 0) {
          const targetRotVel = turnInput * turnSpeed;
          robot.rotVel = applyAccelDecel(robot.rotVel, targetRotVel, turnAccel, dtSec);
        } else {
          robot.rotVel = applyAccelDecel(robot.rotVel, 0, turnDecel, dtSec);
        }

        if (simMatchActive && simKeyboardState.intake) {
          tryIntakeBall(robot);
        }

        if (
          simMatchActive &&
          simKeyboardState.shoot &&
          nowMs - robot.lastShotMs >= shootRepeatMs
        ) {
          tryShootBall(robot, nowMs);
        }

        robot.prevButtons = [];
      } else {
        const activeGamepad = gamepad;
        if (!activeGamepad) {
          robot.velX = applyAccelDecel(robot.velX, 0, decel, dtSec);
          robot.velY = applyAccelDecel(robot.velY, 0, decel, dtSec);
          robot.rotVel = applyAccelDecel(robot.rotVel, 0, turnDecel, dtSec);
          robot.prevButtons = [];
          continue;
        }

        const leftX = applyDeadzone(activeGamepad.axes[0] ?? 0);
        const leftY = applyDeadzone(activeGamepad.axes[1] ?? 0);
        const rightXPrimary = activeGamepad.axes[2] ?? 0;
        const rightXAlt = activeGamepad.axes[3] ?? 0;
        const rotateInput = applyDeadzone(
          Math.abs(rightXPrimary) > Math.abs(rightXAlt) ? rightXPrimary : rightXAlt,
        );

        const targetVelX = leftX * moveSpeed;
        const targetVelY = leftY * moveSpeed;
        const targetRotVel = rotateInput * turnSpeed;

        const moveRate = leftX !== 0 || leftY !== 0 ? accel : decel;
        robot.velX = applyAccelDecel(robot.velX, targetVelX, moveRate, dtSec);
        robot.velY = applyAccelDecel(robot.velY, targetVelY, moveRate, dtSec);

        const rotRate = rotateInput !== 0 ? turnAccel : turnDecel;
        robot.rotVel = applyAccelDecel(robot.rotVel, targetRotVel, rotRate, dtSec);

        const buttons = activeGamepad.buttons.map((btn) => btn.pressed);
        const wasPressed = (idx: number) => !!robot.prevButtons[idx];
        const isPressed = (idx: number) => !!buttons[idx];

        const intakePressed =
          simMatchActive && simIntakeButtonIndexes.some((idx) => isPressed(idx));
        if (intakePressed) {
          tryIntakeBall(robot);
        }

        if (
          simMatchActive &&
          isPressed(simShootButtonIndex) &&
          nowMs - robot.lastShotMs >= shootRepeatMs
        ) {
          tryShootBall(robot, nowMs);
        }

        if (isPressed(simResetButtonIndex) && !wasPressed(simResetButtonIndex)) {
          resetPressed = true;
        }

        robot.prevButtons = buttons;
      }

      robot.xy = {
        x: robot.xy.x + robot.velX * dtSec,
        y: robot.xy.y + robot.velY * dtSec,
      };
      robot.heading += robot.rotVel * dtSec;

      clampPoseToField(robot.xy);
      clampRobotAgainstClassifiers(robot.xy, prevRobotX, prevRobotY);
      updateTurretAimForRobot(robot);
    }

    updateRobotRobotCollision();
    updateBalls(dtMs);
    checkLeverDump(nowMs);

    if (resetPressed) {
      resetSimulatorMatch();
      return;
    }

    // Trigger Svelte reactivity after mutating per-robot state in-place.
    simRobots = [...simRobots];

    if (simRobots[0]) {
      robotXY = { ...simRobots[0].xy };
      robotHeading = simRobots[0].heading;
    }
  }

  const history = createHistory();
  const { canUndoStore, canRedoStore } = history;
  const OPTIMIZER_BASE_URL = "https://fpa.pedropathing.com";

  function getAppState(): AppState {
    return {
      startPoint,
      lines,
      shapes,
      sequence,
      settings,
    };
  }

  // Use the stores for reactivity
  $: canUndo = $canUndoStore;
  $: canRedo = $canRedoStore;

  function recordChange() {
    history.record(getAppState());
  }

  function undoAction() {
    const prev = history.undo();
    if (prev) {
      startPoint = prev.startPoint;
      lines = prev.lines;
      shapes = prev.shapes;
      sequence = prev.sequence;
      settings = prev.settings;
      isUnsaved.set(true);
      two && two.update();
    }

    // undoAction completes; no file-picker behavior here
  }

  function redoAction() {
    const next = history.redo();
    if (next) {
      startPoint = next.startPoint;
      lines = next.lines;
      shapes = next.shapes;
      sequence = next.sequence;
      settings = next.settings;
      isUnsaved.set(true);
      two && two.update();
    }
  }

  $: {
    // Ensure arrays are reactive when items are added/removed
    lines = lines;
    shapes = shapes;
  }

  // Two.js groups
  let lineGroup = new Two.Group();
  lineGroup.id = "line-group";
  let pointGroup = new Two.Group();
  pointGroup.id = "point-group";
  let shapeGroup = new Two.Group();
  shapeGroup.id = "shape-group";
  // Coordinate converters
  let x: d3.ScaleLinear<number, number, number>;

  // Animation controller
  let loopAnimation = true;
  let animationController: ReturnType<typeof createAnimationController>;
  $: timePrediction = calculatePathTime(startPoint, lines, settings, sequence);
  $: animationDuration = getAnimationDuration(timePrediction.totalTime / 1000);
  
  // Second path timeline (for dual path mode)
  $: secondTimePrediction = $dualPathMode && secondStartPoint && secondLines.length > 0 
    ? calculatePathTime(secondStartPoint, secondLines, settings, secondSequence)
    : null;
  
  // Calculate max duration across all paths for playbar scaling
  $: effectiveAnimationDuration = (() => {
    // In multi-path mode, only use additional paths for duration
    if ($activePaths.length > 0) {
      let maxTime = 0;
      additionalPaths.forEach((pathData) => {
        if (pathData.startPoint && pathData.lines.length > 0) {
          const pathTime = calculatePathTime(
            pathData.startPoint,
            pathData.lines,
            pathData.settings,
            pathData.sequence
          );
          if (pathTime) {
            maxTime = Math.max(maxTime, pathTime.totalTime);
          }
        }
      });
      return maxTime > 0 ? getAnimationDuration(maxTime / 1000) : animationDuration;
    }
    
    // In normal/dual mode, check main path and second path
    let maxTime = timePrediction.totalTime;
    
    if ($dualPathMode && secondTimePrediction) {
      maxTime = Math.max(maxTime, secondTimePrediction.totalTime);
    }
    
    return getAnimationDuration(maxTime / 1000);
  })();
  
  // Load additional paths when activePaths changes
  $: {
    loadAdditionalPaths($activePaths);
  }

  async function loadAdditionalPaths(paths: string[]) {
    const newAdditionalPaths: AdditionalPathData[] = [];
    
    // Multi-path mode is isolated - turn off old dual path mode
    if (paths.length > 0) {
      dualPathMode.set(false);
      secondStartPoint = null;
      secondLines = [];
      secondShapes = [];
      secondSequence = [];
      secondFilePath.set(null);
    }
    
    const colors = ['#FF6B6B', '#4ECDC4', '#45B7D1', '#FFA07A']; // Red, Teal, Blue, Salmon

    for (let i = 0; i < Math.min(paths.length, 4); i++) {
      const filePath = paths[i];
      try {
        const content = await browserFileStore.readFile(filePath);
        const data = JSON.parse(content);

        if (data.startPoint && data.lines) {
          const normalizedLines = normalizeLines(data.lines || []);
          newAdditionalPaths.push({
            filePath,
            startPoint: data.startPoint,
            lines: normalizedLines,
            shapes: data.shapes || [],
            sequence: data.sequence || normalizedLines.map((ln: Line) => ({
              kind: "path",
              lineId: ln.id!,
            })),
            settings: data.settings || { ...DEFAULT_SETTINGS },
            color: colors[i],
          });
        }
      } catch (error) {
        console.error(`Failed to load additional path ${filePath}:`, error);
      }
    }

    additionalPaths = newAdditionalPaths;
  }
  
  let secondRobotXY: BasePoint = { x: 0, y: 0 };
  let secondRobotHeading: number = 0;
  /**
   * Converter for X axis from inches to pixels.
   */
  $: x = d3
    .scaleLinear()
    .domain([0, FIELD_SIZE])
    .range([0, width || FIELD_SIZE]);
  /**
   * Converter for Y axis from inches to pixels.
   */
  $: y = d3
    .scaleLinear()
    .domain([0, FIELD_SIZE])
    .range([height || FIELD_SIZE, 0]);
  $: {
    // Calculate robot state using the Timeline
    if (!simulatorMode && timePrediction && timePrediction.timeline && lines.length > 0) {
      const state = calculateRobotState(
        percent,
        timePrediction.timeline,
        lines,
        startPoint,
        settings,
        x,
        y,
      );
      robotXY = { x: state.x, y: state.y };
      robotHeading = state.heading;
    } else {
      // Fallback for initialization
      robotXY = { x: x(startPoint.x), y: y(startPoint.y) };
      robotHeading = 0;
    }
  }

  $: points = (() => {
    if (simulatorMode) {
      return [];
    }

    let _points = [];
    
    // Only show main path points when NOT in multi-path mode
    if ($activePaths.length === 0) {
      let startPointElem = new Two.Circle(
        x(startPoint.x),
        y(startPoint.y),
        x(POINT_RADIUS),
      );
      startPointElem.id = `point-0-0`;
      startPointElem.fill = lines[0].color;
      startPointElem.noStroke();

      _points.push(startPointElem);

      lines.forEach((line, idx) => {
        if (!line || !line.endPoint) return; // Skip invalid lines or lines without endPoint
        [line.endPoint, ...line.controlPoints].forEach((point, idx1) => {
          if (idx1 > 0) {
            let pointGroup = new Two.Group();
            pointGroup.id = `point-${idx + 1}-${idx1}`;

            let pointElem = new Two.Circle(
              x(point.x),
              y(point.y),
              x(POINT_RADIUS),
            );
            pointElem.id = `point-${idx + 1}-${idx1}-background`;
            pointElem.fill = line.color;
            pointElem.noStroke();

            let pointText = new Two.Text(
              `${idx1}`,
              x(point.x),
              y(point.y - 0.15),
              x(POINT_RADIUS),
            );
            pointText.id = `point-${idx + 1}-${idx1}-text`;
            pointText.size = x(1.55);
            pointText.leading = 1;
            pointText.family = "ui-sans-serif, system-ui, sans-serif";
            pointText.alignment = "center";
            pointText.baseline = "middle";
            pointText.fill = "white";
            pointText.noStroke();

            pointGroup.add(pointElem, pointText);
            _points.push(pointGroup);
          } else {
            let pointElem = new Two.Circle(
              x(point.x),
              y(point.y),
              x(POINT_RADIUS),
            );
            pointElem.id = `point-${idx + 1}-${idx1}`;
            pointElem.fill = line.color;
            pointElem.noStroke();
            _points.push(pointElem);
          }
        });
      });
    }
    
    // Add obstacle vertices as draggable points
    shapes.forEach((shape, shapeIdx) => {
      shape.vertices.forEach((vertex, vertexIdx) => {
        let pointGroup = new Two.Group();
        pointGroup.id = `obstacle-${shapeIdx}-${vertexIdx}`;

        let pointElem = new Two.Circle(
          x(vertex.x),
          y(vertex.y),
          x(POINT_RADIUS),
        );
        pointElem.id = `obstacle-${shapeIdx}-${vertexIdx}-background`;
        pointElem.fill = shape.fillColor; // Match obstacle fill color
        pointElem.noStroke();

        let pointText = new Two.Text(
          `${vertexIdx + 1}`,
          x(vertex.x),
          y(vertex.y - 0.15),
          x(POINT_RADIUS),
        );
        pointText.id = `obstacle-${shapeIdx}-${vertexIdx}-text`;
        pointText.size = x(1.55);
        pointText.leading = 1;
        pointText.family = "ui-sans-serif, system-ui, sans-serif";
        pointText.alignment = "center";
        pointText.baseline = "middle";
        pointText.fill = "white";
        pointText.noStroke();
        pointGroup.add(pointElem, pointText);
        _points.push(pointGroup);
      });
    });

    // Add second path points (for dual path mode) - not in multi-path mode
    if ($activePaths.length === 0 && $dualPathMode && secondStartPoint && secondLines.length > 0) {
      let secondStartPointElem = new Two.Circle(
        x(secondStartPoint.x),
        y(secondStartPoint.y),
        x(POINT_RADIUS),
      );
      secondStartPointElem.id = `second-point-0-0`;
      secondStartPointElem.fill = secondLines[0]?.color || "#888";
      secondStartPointElem.noStroke();
      _points.push(secondStartPointElem);

      secondLines.forEach((line, idx) => {
        if (!line || !line.endPoint) return;
        [line.endPoint, ...line.controlPoints].forEach((point, idx1) => {
          if (idx1 > 0) {
            let pointGroup = new Two.Group();
            pointGroup.id = `second-point-${idx + 1}-${idx1}`;

            let pointElem = new Two.Circle(
              x(point.x),
              y(point.y),
              x(POINT_RADIUS),
            );
            pointElem.id = `second-point-${idx + 1}-${idx1}-background`;
            pointElem.fill = line.color;
            pointElem.noStroke();

            let pointText = new Two.Text(
              `${idx1}`,
              x(point.x),
              y(point.y - 0.15),
              x(POINT_RADIUS),
            );
            pointText.id = `second-point-${idx + 1}-${idx1}-text`;
            pointText.size = x(1.55);
            pointText.leading = 1;
            pointText.family = "ui-sans-serif, system-ui, sans-serif";
            pointText.alignment = "center";
            pointText.baseline = "middle";
            pointText.fill = "white";
            pointText.noStroke();

            pointGroup.add(pointElem, pointText);
            _points.push(pointGroup);
          } else {
            let pointElem = new Two.Circle(
              x(point.x),
              y(point.y),
              x(POINT_RADIUS),
            );
            pointElem.id = `second-point-${idx + 1}-${idx1}`;
            pointElem.fill = line.color;
            pointElem.noStroke();
            _points.push(pointElem);
          }
        });
      });
    }

    // Add all control points for additional paths (full editing support)
    if ($activePaths.length > 0) {
      additionalPaths.forEach((pathData, pathIdx) => {
        if (!pathData.startPoint || !pathData.lines.length) return;
        
        // Add starting point
        let startPointElem = new Two.Circle(
          x(pathData.startPoint.x),
          y(pathData.startPoint.y),
          x(POINT_RADIUS * 0.9),
        );
        startPointElem.id = `additional-path-${pathIdx}-point-0-0`;
        startPointElem.fill = pathData.color || pathData.lines[0]?.color || "#888";
        startPointElem.noStroke();
        startPointElem.opacity = 0.8;
        _points.push(startPointElem);
        
        // Add all line points and control points
        pathData.lines.forEach((line, lineIdx) => {
          if (!line || !line.endPoint) return;
          
          [line.endPoint, ...line.controlPoints].forEach((point, pointIdx) => {
            if (pointIdx > 0) {
              // Control point with number
              let pointGroup = new Two.Group();
              pointGroup.id = `additional-path-${pathIdx}-point-${lineIdx + 1}-${pointIdx}`;

              let pointElem = new Two.Circle(
                x(point.x),
                y(point.y),
                x(POINT_RADIUS * 0.9),
              );
              pointElem.id = `additional-path-${pathIdx}-point-${lineIdx + 1}-${pointIdx}-background`;
              pointElem.fill = pathData.color || line.color;
              pointElem.noStroke();

              let pointText = new Two.Text(
                `${pointIdx}`,
                x(point.x),
                y(point.y - 0.15),
                x(POINT_RADIUS * 0.9),
              );
              pointText.id = `additional-path-${pathIdx}-point-${lineIdx + 1}-${pointIdx}-text`;
              pointText.size = x(1.4);
              pointText.leading = 1;
              pointText.family = "ui-sans-serif, system-ui, sans-serif";
              pointText.alignment = "center";
              pointText.baseline = "middle";
              pointText.fill = "white";
              pointText.noStroke();

              pointGroup.add(pointElem, pointText);
              pointGroup.opacity = 0.8;
              _points.push(pointGroup);
            } else {
              // End point without number
              let pointElem = new Two.Circle(
                x(point.x),
                y(point.y),
                x(POINT_RADIUS * 0.9),
              );
              pointElem.id = `additional-path-${pathIdx}-point-${lineIdx + 1}-${pointIdx}`;
              pointElem.fill = pathData.color || line.color;
              pointElem.noStroke();
              pointElem.opacity = 0.8;
              _points.push(pointElem);
            }
          });
        });
      });
    }

    return _points;
  })();

  $: path = (() => {
    if (simulatorMode) {
      return [];
    }

    // Hide main path when in multi-path mode (isolated visualization)
    if ($activePaths.length > 0) {
      return [];
    }
    
    let _path: (Path | PathLine)[] = [];

    lines.forEach((line, idx) => {
      if (!line || !line.endPoint) return; // Skip invalid lines or lines without endPoint
      let _startPoint =
        idx === 0 ? startPoint : lines[idx - 1]?.endPoint || null;
      if (!_startPoint) return; // Skip if previous line's endPoint is missing

      let lineElem: Path | PathLine;
      if (line.controlPoints.length > 2) {
        // Approximate an n-degree bezier curve by sampling it at 100 points
        const samples = 100;
        const cps = [_startPoint, ...line.controlPoints, line.endPoint];
        let points = [
          new Two.Anchor(
            x(_startPoint.x),
            y(_startPoint.y),
            0,
            0,
            0,
            0,
            Two.Commands.move,
          ),
        ];
        for (let i = 1; i <= samples; ++i) {
          const point = getCurvePoint(i / samples, cps);
          points.push(
            new Two.Anchor(
              x(point.x),
              y(point.y),
              0,
              0,
              0,
              0,
              Two.Commands.line,
            ),
          );
        }
        points.forEach((point) => (point.relative = false));
        lineElem = new Two.Path(points);
        lineElem.automatic = false;
      } else if (line.controlPoints.length > 0) {
        let cp1 = line.controlPoints[1]
          ? line.controlPoints[0]
          : quadraticToCubic(_startPoint, line.controlPoints[0], line.endPoint)
              .Q1;
        let cp2 =
          line.controlPoints[1] ??
          quadraticToCubic(_startPoint, line.controlPoints[0], line.endPoint)
            .Q2;
        let points = [
          new Two.Anchor(
            x(_startPoint.x),
            y(_startPoint.y),
            x(_startPoint.x),
            y(_startPoint.y),
            x(cp1.x),
            y(cp1.y),
            Two.Commands.move,
          ),
          new Two.Anchor(
            x(line.endPoint.x),
            y(line.endPoint.y),
            x(cp2.x),
            y(cp2.y),
            x(line.endPoint.x),
            y(line.endPoint.y),
            Two.Commands.curve,
          ),
        ];
        points.forEach((point) => (point.relative = false));

        lineElem = new Two.Path(points);
        lineElem.automatic = false;
      } else {
        lineElem = new Two.Line(
          x(_startPoint.x),
          y(_startPoint.y),
          x(line.endPoint.x),
          y(line.endPoint.y),
        );
      }

      lineElem.id = `line-${idx + 1}`;
      lineElem.stroke = line.color;
      lineElem.linewidth = x(LINE_WIDTH);
      lineElem.noFill();
      // Add a dashed line for locked paths
      const baseOpacity = settings.pathOpacity || 1.0;
      if (line.locked) {
        lineElem.dashes = [x(2), x(2)];
        lineElem.opacity = baseOpacity * 0.7;
      } else {
        lineElem.dashes = [];
        lineElem.opacity = baseOpacity;
      }

      _path.push(lineElem);
    });

    return _path;
  })();

  // Second path rendering (for dual path mode)
  $: secondPath = (() => {
    if (simulatorMode) {
      return [];
    }

    // Don't show second path when in multi-path mode (use activePaths instead)
    if ($activePaths.length > 0 || !$dualPathMode || !secondStartPoint || secondLines.length === 0) {
      return [];
    }

    let _path: (Path | PathLine)[] = [];

    secondLines.forEach((line, idx) => {
      if (!line || !line.endPoint) return;
      let _startPoint =
        idx === 0 ? secondStartPoint : secondLines[idx - 1]?.endPoint || null;
      if (!_startPoint) return;

      let lineElem: Path | PathLine;
      if (line.controlPoints.length > 2) {
        const samples = 100;
        const cps = [_startPoint, ...line.controlPoints, line.endPoint];
        let points = [
          new Two.Anchor(
            x(_startPoint.x),
            y(_startPoint.y),
            0,
            0,
            0,
            0,
            Two.Commands.move,
          ),
        ];
        for (let i = 1; i <= samples; ++i) {
          const point = getCurvePoint(i / samples, cps);
          points.push(
            new Two.Anchor(
              x(point.x),
              y(point.y),
              0,
              0,
              0,
              0,
              Two.Commands.line,
            ),
          );
        }
        points.forEach((point) => (point.relative = false));
        lineElem = new Two.Path(points);
        lineElem.automatic = false;
      } else if (line.controlPoints.length > 0) {
        let cp1 = line.controlPoints[1]
          ? line.controlPoints[0]
          : quadraticToCubic(_startPoint, line.controlPoints[0], line.endPoint)
              .Q1;
        let cp2 =
          line.controlPoints[1] ??
          quadraticToCubic(_startPoint, line.controlPoints[0], line.endPoint)
            .Q2;
        let points = [
          new Two.Anchor(
            x(_startPoint.x),
            y(_startPoint.y),
            x(_startPoint.x),
            y(_startPoint.y),
            x(cp1.x),
            y(cp1.y),
            Two.Commands.move,
          ),
          new Two.Anchor(
            x(line.endPoint.x),
            y(line.endPoint.y),
            x(cp2.x),
            y(cp2.y),
            x(line.endPoint.x),
            y(line.endPoint.y),
            Two.Commands.curve,
          ),
        ];
        points.forEach((point) => (point.relative = false));

        lineElem = new Two.Path(points);
        lineElem.automatic = false;
      } else {
        lineElem = new Two.Line(
          x(_startPoint.x),
          y(_startPoint.y),
          x(line.endPoint.x),
          y(line.endPoint.y),
        );
      }

      lineElem.id = `second-line-${idx + 1}`;
      lineElem.stroke = line.color;
      lineElem.linewidth = x(LINE_WIDTH);
      lineElem.noFill();
      const baseOpacity = settings.pathOpacity || 1.0;
      if (line.locked) {
        lineElem.dashes = [x(2), x(2)];
        lineElem.opacity = baseOpacity * 0.7;
      } else {
        lineElem.dashes = [];
        lineElem.opacity = baseOpacity;
      }

      _path.push(lineElem);
    });

    return _path;
  })();

  // Render all additional paths
  $: additionalPathElements = additionalPaths.map((pathData, pathIdx) => {
    if (simulatorMode) {
      return [];
    }

    if (!pathData.startPoint || pathData.lines.length === 0) {
      return [];
    }

    let _path: (Path | PathLine)[] = [];
    // All paths should be clearly visible - only slight opacity variation
    const pathOpacityBase = settings.pathOpacity || 1.0;
    const opacity = (1.0 - (pathIdx * 0.1)) * pathOpacityBase;

    pathData.lines.forEach((line, idx) => {
      if (!line || !line.endPoint) return;
      let _startPoint =
        idx === 0 ? pathData.startPoint : pathData.lines[idx - 1]?.endPoint || null;
      if (!_startPoint) return;

      let lineElem: Path | PathLine;
      if (line.controlPoints.length > 2) {
        const samples = 100;
        const cps = [_startPoint, ...line.controlPoints, line.endPoint];
        let points = [
          new Two.Anchor(
            x(_startPoint.x),
            y(_startPoint.y),
            0,
            0,
            0,
            0,
            Two.Commands.move,
          ),
        ];
        for (let i = 1; i <= samples; ++i) {
          const point = getCurvePoint(i / samples, cps);
          points.push(
            new Two.Anchor(
              x(point.x),
              y(point.y),
              0,
              0,
              0,
              0,
              Two.Commands.line,
            ),
          );
        }
        points.forEach((point) => (point.relative = false));
        lineElem = new Two.Path(points);
        lineElem.automatic = false;
      } else if (line.controlPoints.length > 0) {
        let cp1 = line.controlPoints[1]
          ? line.controlPoints[0]
          : quadraticToCubic(_startPoint, line.controlPoints[0], line.endPoint)
              .Q1;
        let cp2 =
          line.controlPoints[1] ??
          quadraticToCubic(_startPoint, line.controlPoints[0], line.endPoint)
            .Q2;
        let points = [
          new Two.Anchor(
            x(_startPoint.x),
            y(_startPoint.y),
            x(_startPoint.x),
            y(_startPoint.y),
            x(cp1.x),
            y(cp1.y),
            Two.Commands.move,
          ),
          new Two.Anchor(
            x(line.endPoint.x),
            y(line.endPoint.y),
            x(cp2.x),
            y(cp2.y),
            x(line.endPoint.x),
            y(line.endPoint.y),
            Two.Commands.curve,
          ),
        ];
        points.forEach((point) => (point.relative = false));

        lineElem = new Two.Path(points);
        lineElem.automatic = false;
      } else {
        lineElem = new Two.Line(
          x(_startPoint.x),
          y(_startPoint.y),
          x(line.endPoint.x),
          y(line.endPoint.y),
        );
      }

      lineElem.id = `additional-path-${pathIdx}-line-${idx + 1}`;
      lineElem.stroke = pathData.color || line.color;
      lineElem.linewidth = x(LINE_WIDTH);
      lineElem.noFill();
      lineElem.opacity = opacity;

      _path.push(lineElem);
    });

    return _path;
  });

  $: shapeElements = (() => {
    if (simulatorMode) {
      return [];
    }

    // Obstacles removed: return empty array for shape elements
    let _shapes: Path[] = [];

    shapes.forEach((shape, idx) => {
      if (shape.vertices.length >= 3) {
        // Create polygon from vertices - properly format for Two.js
        let vertices = [];

        // Start with move command for first vertex
        vertices.push(
          new Two.Anchor(
            x(shape.vertices[0].x),
            y(shape.vertices[0].y),
            0,
            0,
            0,
            0,
            Two.Commands.move,
          ),
        );

        // Add line commands for remaining vertices
        for (let i = 1; i < shape.vertices.length; i++) {
          vertices.push(
            new Two.Anchor(
              x(shape.vertices[i].x),
              y(shape.vertices[i].y),
              0,
              0,
              0,
              0,
              Two.Commands.line,
            ),
          );
        }

        // Close the shape
        vertices.push(
          new Two.Anchor(
            x(shape.vertices[0].x),
            y(shape.vertices[0].y),
            0,
            0,
            0,
            0,
            Two.Commands.close,
          ),
        );

        vertices.forEach((point) => (point.relative = false));

        let shapeElement = new Two.Path(vertices);
        shapeElement.id = `shape-${idx}`;
        shapeElement.stroke = shape.color;
        shapeElement.fill = shape.color;
        shapeElement.opacity = 0.4;
        shapeElement.linewidth = x(0.8);
        shapeElement.automatic = false;

        _shapes.push(shapeElement);
      }
    });

    return _shapes;
  })();

  $: ghostPathElement = (() => {
    if (simulatorMode) {
      return null;
    }

    let ghostPath: Path | null = null;

    // Don't show ghost paths in multi-path mode
    if ($activePaths.length === 0 && settings.showGhostPaths && lines.length > 0) {
      const ghostPoints = generateGhostPathPoints(
        startPoint,
        lines,
        settings.rWidth,
        settings.rHeight,
        50,
      );

      if (ghostPoints.length >= 3) {
        // Create polygon from ghost path points
        let vertices = [];

        // Start with move command for first point
        vertices.push(
          new Two.Anchor(
            x(ghostPoints[0].x),
            y(ghostPoints[0].y),
            0,
            0,
            0,
            0,
            Two.Commands.move,
          ),
        );

        // Add line commands for remaining points
        for (let i = 1; i < ghostPoints.length; i++) {
          vertices.push(
            new Two.Anchor(
              x(ghostPoints[i].x),
              y(ghostPoints[i].y),
              0,
              0,
              0,
              0,
              Two.Commands.line,
            ),
          );
        }

        // Close the shape
        vertices.push(
          new Two.Anchor(
            x(ghostPoints[0].x),
            y(ghostPoints[0].y),
            0,
            0,
            0,
            0,
            Two.Commands.close,
          ),
        );

        vertices.forEach((point) => (point.relative = false));

        ghostPath = new Two.Path(vertices);
        ghostPath.id = "ghost-path";
        ghostPath.stroke = "#a78bfa"; // Light purple/lavender
        ghostPath.fill = "#a78bfa";
        ghostPath.opacity = 0.15;
        ghostPath.linewidth = x(0.5);
        ghostPath.automatic = false;
      }
    }

    return ghostPath;
  })();

  // Second ghost path for dual path mode
  $: secondGhostPathElement = (() => {
    if (simulatorMode) {
      return null;
    }

    let ghostPath: Path | null = null;

    // Don't show second ghost path in multi-path mode
    if ($activePaths.length === 0 && $dualPathMode && settings.showGhostPaths && secondLines.length > 0 && secondStartPoint) {
      const ghostPoints = generateGhostPathPoints(
        secondStartPoint,
        secondLines,
        settings.rWidth,
        settings.rHeight,
        50,
      );

      if (ghostPoints.length >= 3) {
        let vertices = [];

        vertices.push(
          new Two.Anchor(
            x(ghostPoints[0].x),
            y(ghostPoints[0].y),
            0,
            0,
            0,
            0,
            Two.Commands.move,
          ),
        );

        for (let i = 1; i < ghostPoints.length; i++) {
          vertices.push(
            new Two.Anchor(
              x(ghostPoints[i].x),
              y(ghostPoints[i].y),
              0,
              0,
              0,
              0,
              Two.Commands.line,
            ),
          );
        }

        vertices.push(
          new Two.Anchor(
            x(ghostPoints[0].x),
            y(ghostPoints[0].y),
            0,
            0,
            0,
            0,
            Two.Commands.close,
          ),
        );

        vertices.forEach((point) => (point.relative = false));

        ghostPath = new Two.Path(vertices);
        ghostPath.id = "ghost-path-2";
        ghostPath.stroke = "#fca5a5"; // Light red/pink for second robot
        ghostPath.fill = "#fca5a5";
        ghostPath.opacity = 0.15;
        ghostPath.linewidth = x(0.5);
        ghostPath.automatic = false;
      }
    }

    return ghostPath;
  })();

  // Ghost paths for additional paths in multi-path mode
  $: additionalGhostPathElements = (() => {
    if (simulatorMode) {
      return [];
    }

    let ghostPaths: Path[] = [];

    if ($activePaths.length > 0 && settings.showGhostPaths) {
      additionalPaths.forEach((pathData, pathIdx) => {
        if (!pathData.startPoint || !pathData.lines.length) return;

        const ghostPoints = generateGhostPathPoints(
          pathData.startPoint,
          pathData.lines,
          settings.rWidth,
          settings.rHeight,
          50,
        );

        if (ghostPoints.length >= 3) {
          let vertices = [];

          vertices.push(
            new Two.Anchor(
              x(ghostPoints[0].x),
              y(ghostPoints[0].y),
              0,
              0,
              0,
              0,
              Two.Commands.move,
            ),
          );

          for (let i = 1; i < ghostPoints.length; i++) {
            vertices.push(
              new Two.Anchor(
                x(ghostPoints[i].x),
                y(ghostPoints[i].y),
                0,
                0,
                0,
                0,
                Two.Commands.line,
              ),
            );
          }

          vertices.push(
            new Two.Anchor(
              x(ghostPoints[0].x),
              y(ghostPoints[0].y),
              0,
              0,
              0,
              0,
              Two.Commands.close,
            ),
          );

          vertices.forEach((point) => (point.relative = false));

          const ghostPath = new Two.Path(vertices);
          ghostPath.id = `ghost-path-additional-${pathIdx}`;
          ghostPath.stroke = pathData.color || "#a78bfa";
          ghostPath.fill = pathData.color || "#a78bfa";
          ghostPath.opacity = 0.15;
          ghostPath.linewidth = x(0.5);
          ghostPath.automatic = false;
          
          ghostPaths.push(ghostPath);
        }
      });
    }

    return ghostPaths;
  })();

  $: onionLayerElements = (() => {
    if (simulatorMode) {
      return [];
    }

    let onionLayers: Path[] = [];

    // Don't show onion layers in multi-path mode
    if ($activePaths.length === 0 && settings.showOnionLayers && lines.length > 0) {
      const spacing = settings.onionLayerSpacing || 6;
      let layers = generateOnionLayers(
        startPoint,
        lines,
        settings.rWidth,
        settings.rHeight,
        spacing,
      );

      // If user requested onion layers only for the next point, filter to the relevant line
      if (
        settings.onionNextPointOnly &&
        timePrediction &&
        timePrediction.timeline
      ) {
        const currentTime = (timePrediction.totalTime || 0) * (percent / 100);
        const travelEvents = (timePrediction.timeline || []).filter(
          (ev) => ev.type === "travel",
        );

        let selectedLineIndex: number | null = null;

        // Current travel segment
        const currentTravel = travelEvents.find(
          (ev) => ev.startTime <= currentTime && ev.endTime >= currentTime,
        );
        if (currentTravel) {
          selectedLineIndex = currentTravel.lineIndex as number;
        } else {
          // Next upcoming travel segment
          const nextTravel = travelEvents.find(
            (ev) => ev.startTime > currentTime,
          );
          if (nextTravel) selectedLineIndex = nextTravel.lineIndex as number;
          else if (travelEvents.length)
            selectedLineIndex = travelEvents[travelEvents.length - 1]
              .lineIndex as number;
        }

        if (selectedLineIndex !== null) {
          layers = layers.filter((l: any) => l.lineIndex === selectedLineIndex);
        }
      }

      layers.forEach((layer, idx) => {
        // Create a rectangle from the robot corners
        let vertices: any[] = [];

        // Create path from corners: front-left -> front-right -> back-right -> back-left
        vertices.push(
          new Two.Anchor(
            x(layer.corners[0].x),
            y(layer.corners[0].y),
            0,
            0,
            0,
            0,
            Two.Commands.move,
          ),
        );

        for (let i = 1; i < layer.corners.length; i++) {
          vertices.push(
            new Two.Anchor(
              x(layer.corners[i].x),
              y(layer.corners[i].y),
              0,
              0,
              0,
              0,
              Two.Commands.line,
            ),
          );
        }

        // Close the path by returning to the first corner
        vertices.push(
          new Two.Anchor(
            x(layer.corners[0].x),
            y(layer.corners[0].y),
            0,
            0,
            0,
            0,
            Two.Commands.close,
          ),
        );

        vertices.forEach((point) => (point.relative = false));

        let onionRect = new Two.Path(vertices);
        onionRect.id = `onion-layer-${idx}`;
        onionRect.stroke = settings.onionColor || "#dc2626";
        onionRect.noFill();
        // Increase opacity so colliders are more visible
        onionRect.opacity = 0.9;
        onionRect.linewidth = x(0.28);
        onionRect.automatic = false;

        onionLayers.push(onionRect);
      });
    }

    return onionLayers;
  })();

  // Second onion layers for dual path mode
  $: secondOnionLayerElements = (() => {
    if (simulatorMode) {
      return [];
    }

    let onionLayers: Path[] = [];

    // Don't show second onion layers in multi-path mode
    if ($activePaths.length === 0 && $dualPathMode && settings.showOnionLayers && secondLines.length > 0 && secondStartPoint) {
      const spacing = settings.onionLayerSpacing || 6;
      let layers = generateOnionLayers(
        secondStartPoint,
        secondLines,
        settings.rWidth,
        settings.rHeight,
        spacing,
      );

      // If user requested onion layers only for the next point, filter to the relevant line
      if (
        settings.onionNextPointOnly &&
        secondTimePrediction &&
        secondTimePrediction.timeline
      ) {
        const currentTime = (secondTimePrediction.totalTime || 0) * (percent / 100);
        const travelEvents = (secondTimePrediction.timeline || []).filter(
          (ev) => ev.type === "travel",
        );

        let selectedLineIndex: number | null = null;

        const currentTravel = travelEvents.find(
          (ev) => ev.startTime <= currentTime && ev.endTime >= currentTime,
        );
        if (currentTravel) {
          selectedLineIndex = currentTravel.lineIndex as number;
        } else {
          const nextTravel = travelEvents.find(
            (ev) => ev.startTime > currentTime,
          );
          if (nextTravel) selectedLineIndex = nextTravel.lineIndex as number;
          else if (travelEvents.length)
            selectedLineIndex = travelEvents[travelEvents.length - 1]
              .lineIndex as number;
        }

        if (selectedLineIndex !== null) {
          layers = layers.filter((l: any) => l.lineIndex === selectedLineIndex);
        }
      }

      layers.forEach((layer, idx) => {
        let vertices: any[] = [];

        vertices.push(
          new Two.Anchor(
            x(layer.corners[0].x),
            y(layer.corners[0].y),
            0,
            0,
            0,
            0,
            Two.Commands.move,
          ),
        );

        for (let i = 1; i < layer.corners.length; i++) {
          vertices.push(
            new Two.Anchor(
              x(layer.corners[i].x),
              y(layer.corners[i].y),
              0,
              0,
              0,
              0,
              Two.Commands.line,
            ),
          );
        }

        vertices.push(
          new Two.Anchor(
            x(layer.corners[0].x),
            y(layer.corners[0].y),
            0,
            0,
            0,
            0,
            Two.Commands.close,
          ),
        );

        vertices.forEach((point) => (point.relative = false));

        let onionRect = new Two.Path(vertices);
        onionRect.id = `second-onion-layer-${idx}`;
        onionRect.stroke = "#fca5a5"; // Light red/pink for second path
        onionRect.noFill();
        onionRect.opacity = 0.9;
        onionRect.linewidth = x(0.28);
        onionRect.automatic = false;

        onionLayers.push(onionRect);
      });
    }

    return onionLayers;
  })();

  let isLoaded = false;
  // Reactively trigger when any saveable data changes
  $: {
    if (isLoaded && (lines || shapes || startPoint || settings)) {
      isUnsaved.set(true);
    }
  }

  // Allow the app to stabilize before tracking changes
  onMount(() => {
    setTimeout(() => {
      isLoaded = true;
      recordChange();
    }, 500);
  });
  onMount(async () => {
    // Load saved settings
    const savedSettings = await loadSettings();
    settings = { ...savedSettings };

    // Update robot dimensions from loaded settings
    robotWidth = settings.rWidth;
    robotHeight = settings.rHeight;
  });
  // Debounced save function
  const debouncedSaveSettings = debounce(async (settingsToSave: Settings) => {
    await saveSettings(settingsToSave);
  }, 1000);
  // Save after 1 second of inactivity

  // Watch for settings changes and save
  $: {
    if (settings) {
      debouncedSaveSettings(settings);
    }
  }

  // Initialize animation controller
  onMount(() => {
    animationController = createAnimationController(
      animationDuration,
      (newPercent) => {
        percent = newPercent;
      },
      () => {
        // Animation completed callback
        console.log("Animation completed");
        playing = false;
      },
    );
  });
  $: if (animationController) {
    animationController.setDuration(effectiveAnimationDuration);
  }

  $: if (animationController) {
    animationController.setLoop(loopAnimation);
    // Sync UI state with controller
    playing = animationController.isPlaying();
  }

  // Save Function
  // Save the current project into the browser-backed store (or download)
  async function saveProject() {
    try {
      await saveFile();
    } catch (e) {
      console.error("Failed to save project:", e);
      alert("Failed to save file.");
    }
  }

  // Save an additional path back to its file
  async function saveAdditionalPath(pathIdx: number) {
    const pathData = additionalPaths[pathIdx];
    if (!pathData || !pathData.filePath) return;
    
    try {
      const fileData = JSON.stringify({
        startPoint: pathData.startPoint,
        lines: pathData.lines,
        shapes: pathData.shapes,
        sequence: pathData.sequence,
        settings: pathData.settings,
        version: "1.2.1",
        timestamp: new Date().toISOString(),
      });
      
      await browserFileStore.writeFile(pathData.filePath, fileData);
      console.log(`Auto-saved additional path: ${pathData.filePath}`);
    } catch (error) {
      console.error(`Failed to save additional path ${pathData.filePath}:`, error);
      throw error;
    }
  }

  // Keyboard shortcut for save
  hotkeys("cmd+s, ctrl+s", function (event, handler) {
    event.preventDefault();
    if ($activePaths.length > 0) {
      // Multiple paths mode - save all modified paths
      showDualPathSaveDialog = true;
    } else if ($dualPathMode && secondStartPoint && secondLines.length > 0) {
      showDualPathSaveDialog = true;
    } else {
      showSaveDialog = true;
    }
  });
  $: {
    // This handles both 'travel' (movement) and 'wait' (stationary rotation) events.
    // Don't show main robot in multi-path mode
    if (
      !simulatorMode &&
      $activePaths.length === 0 &&
      timePrediction &&
      timePrediction.timeline &&
      lines.length > 0
    ) {
      const state = calculateRobotState(
        percent,
        timePrediction.timeline,
        lines,
        startPoint,
        settings,
        x,
        y,
      );
      robotXY = { x: state.x, y: state.y };
      robotHeading = state.heading;
    } else {
      // Fallback for initialization or empty state
      robotXY = { x: x(startPoint.x), y: y(startPoint.y) };
      // Calculate initial heading based on start point settings
      if (startPoint.heading === "linear") robotHeading = -startPoint.startDeg;
      else if (startPoint.heading === "constant")
        robotHeading = -startPoint.degrees;
      else robotHeading = 0;
    }
  }

  // Second robot state calculation (for dual path mode)
  $: {
    // Don't show second robot in multi-path mode
    if (
      !simulatorMode &&
      $activePaths.length === 0 &&
      $dualPathMode &&
      timePrediction &&
      secondTimePrediction &&
      secondTimePrediction.timeline &&
      secondLines.length > 0 &&
      secondStartPoint
    ) {
      // Calculate actual percent for this path based on max duration
      const maxDuration = effectiveAnimationDuration;
      const thisDuration = getAnimationDuration(secondTimePrediction.totalTime / 1000);
      const completionPercent = (thisDuration / maxDuration) * 100;
      
      // If this path should be complete, cap at 100% (robot waits at end)
      const actualPercent = Math.min(percent, completionPercent);
      const normalizedPercent = completionPercent > 0 ? (actualPercent / completionPercent) * 100 : 0;

      const state = calculateRobotState(
        normalizedPercent,
        secondTimePrediction.timeline,
        secondLines,
        secondStartPoint,
        settings,
        x,
        y,
      );
      secondRobotXY = { x: state.x, y: state.y };
      secondRobotHeading = state.heading;
    } else {
      // Fallback or not in dual mode
      secondRobotXY = { x: 0, y: 0 };
      secondRobotHeading = 0;
    }
  }

  $: {
    if (simulatorMode && width > 0 && height > 0 && !simInitialized) {
      resetSimulatorMatch();
      simInitialized = true;
    }
  }

  $: {
    if (simulatorMode && width > 0 && height > 0 && simInitialized) {
      handleFieldResizeForSimulator();
      simRobots.forEach((robot) => clampPoseToField(robot.xy));
    }
  }

  onMount(() => {
    if (!simulatorMode) return;

    const onGamepadConnected = () => refreshControllerStatus();
    const onGamepadDisconnected = () => refreshControllerStatus();
    const onKeyDown = (event: KeyboardEvent) => handleSimulatorKeyboard(event, true);
    const onKeyUp = (event: KeyboardEvent) => handleSimulatorKeyboard(event, false);
    const onWindowBlur = () => clearSimKeyboardState();
    refreshControllerStatus();

    const frame = (timestamp: number) => {
      if (!simLastFrameTime) {
        simLastFrameTime = timestamp;
      }
      const dt = Math.min(40, timestamp - simLastFrameTime);
      simLastFrameTime = timestamp;
      updateSimulator(dt);
      simAnimationFrame = requestAnimationFrame(frame);
    };

    window.addEventListener("gamepadconnected", onGamepadConnected);
    window.addEventListener("gamepaddisconnected", onGamepadDisconnected);
    window.addEventListener("keydown", onKeyDown);
    window.addEventListener("keyup", onKeyUp);
    window.addEventListener("blur", onWindowBlur);
    simAnimationFrame = requestAnimationFrame(frame);

    return () => {
      window.removeEventListener("gamepadconnected", onGamepadConnected);
      window.removeEventListener("gamepaddisconnected", onGamepadDisconnected);
      window.removeEventListener("keydown", onKeyDown);
      window.removeEventListener("keyup", onKeyUp);
      window.removeEventListener("blur", onWindowBlur);
      if (simAnimationFrame) {
        cancelAnimationFrame(simAnimationFrame);
      }
    };
  });

  // Calculate robot states for all additional paths
  let additionalRobotStates: Array<{ xy: BasePoint; heading: number }> = [];
  $: {
    additionalRobotStates = additionalPaths.map((pathData) => {
      if (!pathData.startPoint) {
        return {
          xy: { x: 0, y: 0 },
          heading: 0,
        };
      }

      const pathTimePrediction = calculatePathTime(
        pathData.startPoint,
        pathData.lines,
        pathData.settings,
        pathData.sequence
      );
      
      if (pathTimePrediction && pathTimePrediction.timeline && pathData.lines.length > 0 && pathData.startPoint) {
        // Calculate actual percent for this path based on max duration
        const maxDuration = effectiveAnimationDuration;
        const thisDuration = getAnimationDuration(pathTimePrediction.totalTime / 1000);
        const completionPercent = (thisDuration / maxDuration) * 100;
        
        // If this path should be complete, cap at 100% (robot waits at end)
        const actualPercent = Math.min(percent, completionPercent);
        const normalizedPercent = completionPercent > 0 ? (actualPercent / completionPercent) * 100 : 0;

        const state = calculateRobotState(
          normalizedPercent,
          pathTimePrediction.timeline,
          pathData.lines,
          pathData.startPoint,
          pathData.settings,
          x,
          y,
        );
        
        return {
          xy: { x: state.x, y: state.y },
          heading: state.heading,
        };
      }
      
      return {
        xy: { x: 0, y: 0 },
        heading: 0,
      };
    });
  }

  // Event markers removed: no runtime visualization created

  $: (() => {
    if (!two) {
      return;
    }

    two.renderer.domElement.style["z-index"] = "30";
    two.renderer.domElement.style["position"] = "absolute";
    two.renderer.domElement.style["top"] = "0px";
    two.renderer.domElement.style["left"] = "0px";
    two.renderer.domElement.style["width"] = "100%";
    two.renderer.domElement.style["height"] = "100%";

    two.clear();

    two.add(...shapeElements);
    if (ghostPathElement) {
      two.add(ghostPathElement);
    }
    if (secondGhostPathElement) {
      two.add(secondGhostPathElement);
    }
    if (additionalGhostPathElements.length > 0) {
      two.add(...additionalGhostPathElements);
    }
    if (onionLayerElements.length > 0) {
      two.add(...onionLayerElements);
    }
    if (secondOnionLayerElements.length > 0) {
      two.add(...secondOnionLayerElements);
    }
    two.add(...path);
    if ($dualPathMode && secondPath.length > 0) {
      two.add(...secondPath);
    }
    // Add all additional paths
    if ($activePaths.length > 0) {
      additionalPathElements.forEach((pathElements) => {
        if (pathElements.length > 0) {
          two.add(...pathElements);
        }
      });
    }
    two.add(...points);

    two.update();
  })();
  async function saveFileAs() {
    const win: any = window as any;
    const content = JSON.stringify(
      {
        startPoint,
        lines,
        shapes,
        sequence,
        settings,
        version: "1.2.1",
        timestamp: new Date().toISOString(),
      },
      null,
      2,
    );

    // Prefer File System Access API if available: opens native Save dialog
    if (win.showSaveFilePicker) {
      try {
        const opts = {
          suggestedName: $currentFilePath
            ? $currentFilePath.split(/[\/]/).pop()
            : "path.pp",
          types: [
            {
              description: "Path files",
              accept: { "application/json": [".pp", ".json"] },
            },
          ],
        };

        const handle = await win.showSaveFilePicker(opts);
        if (!handle) {
          // User cancelled
          return;
        }

        const writable = await handle.createWritable();
        await writable.write(content);
        await writable.close();

        // Update app state to reflect saved file
        try {
          currentFilePath.set(
            handle.name || (typeof handle === "string" ? handle : null),
          );
        } catch (e) {
          // ignore
        }
        isUnsaved.set(false);
        alert(`Saved to: ${handle.name || "selected file"}`);
        return;
      } catch (err) {
        console.error("SaveFilePicker error:", err);
        // fall through to download fallback
      }
    }

    // If showSaveFilePicker is not available or failed, try showOpenFilePicker to let user pick an existing file to overwrite
    if (win.showOpenFilePicker) {
      try {
        const [handle] = await win.showOpenFilePicker({
          types: [
            {
              description: "Path files",
              accept: { "application/json": [".pp", ".json"] },
            },
          ],
          multiple: false,
        });

        if (handle) {
          const writable = await handle.createWritable();
          await writable.write(content);
          await writable.close();
          try {
            currentFilePath.set(handle.name || null);
          } catch (e) {}
          isUnsaved.set(false);
          alert(`Saved to local file: ${handle.name || "selected file"}`);
          return;
        }
      } catch (err) {
        console.error("showOpenFilePicker error:", err);
        // fall through to download fallback
      }
    }

    // Fallback for browsers without File System Access (e.g., Firefox).
    // Automatically save into the app's browser-backed storage to avoid forcing a download.
    try {
      await saveFile();
      alert(
        "Your browser does not support native file dialogs. The project was saved to the app's storage.\n\nOpen the File Manager to download or export the file to your computer.",
      );
    } catch (err) {
      console.error("Failed to save into app storage:", err);
      // As a last resort, download the file
      try {
        downloadTrajectory(startPoint, lines, shapes, sequence);
      } catch (err2) {
        console.error("Save As fallback failed:", err2);
        alert(
          "Failed to save file. Your browser may not support file picker APIs.",
        );
      }
    }
  }

  function animate(timestamp: number) {
    if (!startTime) {
      startTime = timestamp;
    }

    if (previousTime !== null) {
      const deltaTime = timestamp - previousTime;
      if (percent >= 100) {
        percent = 0;
      } else {
        percent += (0.65 / lines.length) * (deltaTime * 0.1);
      }
    }

    previousTime = timestamp;

    if (playing) {
      requestAnimationFrame(animate);
    }
  }

  function play() {
    animationController.play();
    playing = true;
  }

  function pause() {
    animationController.pause();
    playing = false;
  }

  function resetAnimation() {
    animationController.reset();
    playing = false;
  }

  // Handle slider changes
  function handleSeek(newPercent: number) {
    if (animationController) {
      animationController.seekToPercent(newPercent);
    }
  }

  onMount(() => {
    two = new Two({
      fitted: true,
      type: Two.Types.svg,
    }).appendTo(twoElement);

    updateRobotImageDisplay();

    let currentElem: string | null = null;
    let isDown = false;
    let dragOffset = { x: 0, y: 0 }; // Store offset to prevent snapping to center

    const isLockedPathElem = (id: string | null): boolean => {
      if (!id || !id.startsWith("point")) return false;
      const parts = id.split("-");
      const lineIdx = Number(parts[1]) - 1;
      if (Number.isNaN(lineIdx)) return false;
      if (lineIdx < 0) return false; // startPoint currently not lockable
      return !!lines[lineIdx]?.locked;
    };

    two.renderer.domElement.addEventListener("mousemove", (evt: MouseEvent) => {
      if (simulatorMode) {
        two.renderer.domElement.style.cursor = "auto";
        return;
      }

      const elem = document.elementFromPoint(evt.clientX, evt.clientY);

      if (isDown && currentElem) {
        const parts = currentElem.split("-");
        const isPathPoint = parts[0] === "point";
        const isShapePoint = parts[0] === "shape";

        // Skip dragging locked paths
        if (isPathPoint) {
          const hitLine = Number(parts[1]) - 1;
          if (hitLine >= 0 && lines[hitLine]?.locked) return;
        }

        // Use simple bounding rect math to match D3 scales which are bound to clientWidth/Height
        const rect = two.renderer.domElement.getBoundingClientRect();
        const xPos = evt.clientX - rect.left;
        const yPos = evt.clientY - rect.top;

        // Get current store values for reactivity
        const currentGridSize = $gridSize;
        const currentSnapToGrid = $snapToGrid;
        const currentShowGrid = $showGrid;

        // Apply drag offset (in inches) to the raw mouse position
        let rawInchX = x.invert(xPos) + dragOffset.x;
        let rawInchY = y.invert(yPos) + dragOffset.y;

        let inchX = rawInchX;
        let inchY = rawInchY;

        // Always apply grid snapping when enabled
        if (currentSnapToGrid && currentShowGrid && currentGridSize > 0) {
          // Force snap to nearest grid point
          inchX = Math.round(rawInchX / currentGridSize) * currentGridSize;
          inchY = Math.round(rawInchY / currentGridSize) * currentGridSize;

          // Clamp to field boundaries
          inchX = Math.max(0, Math.min(FIELD_SIZE, inchX));
          inchY = Math.max(0, Math.min(FIELD_SIZE, inchY));
        }

        // Handle path point dragging
        if (currentElem.startsWith("obstacle-")) {
          // Handle obstacle vertex dragging
          const parts = currentElem.split("-");
          const shapeIdx = Number(parts[1]);
          const vertexIdx = Number(parts[2]);

          shapes[shapeIdx].vertices[vertexIdx].x = inchX;
          shapes[shapeIdx].vertices[vertexIdx].y = inchY;
          shapes = [...shapes];
        } else if (currentElem.startsWith("second-point-")) {
          // Handle second path point dragging
          const parts = currentElem.split("-");
          const line = Number(parts[2]) - 1;
          const point = Number(parts[3]);

          if (line === -1) {
            // This is the second starting point
            if (secondStartPoint?.locked) return;
            if (secondStartPoint) {
              secondStartPoint.x = inchX;
              secondStartPoint.y = inchY;
            }
          } else if (secondLines[line]) {
            if (point === 0 && secondLines[line].endPoint) {
              secondLines[line].endPoint.x = inchX;
              secondLines[line].endPoint.y = inchY;
            } else {
              if (secondLines[line]?.locked) return;
              secondLines[line].controlPoints[point - 1].x = inchX;
              secondLines[line].controlPoints[point - 1].y = inchY;
            }
          }
          secondLines = [...secondLines];
        } else if (currentElem.startsWith("additional-path-")) {
          // Handle additional path point dragging
          const parts = currentElem.split("-");
          const pathIdx = Number(parts[2]);
          const line = Number(parts[4]) - 1;
          const point = Number(parts[5]);

          if (!additionalPaths[pathIdx]) return;

          if (line === -1) {
            // This is the starting point
            if (additionalPaths[pathIdx].startPoint) {
              additionalPaths[pathIdx].startPoint.x = inchX;
              additionalPaths[pathIdx].startPoint.y = inchY;
              additionalPaths = [...additionalPaths];
              // Auto-save changes to additional path files
              saveAdditionalPath(pathIdx).catch(err => console.error('Failed to auto-save additional path:', err));
            }
          } else if (additionalPaths[pathIdx].lines[line]) {
            if (point === 0 && additionalPaths[pathIdx].lines[line].endPoint) {
              additionalPaths[pathIdx].lines[line].endPoint.x = inchX;
              additionalPaths[pathIdx].lines[line].endPoint.y = inchY;
            } else if (additionalPaths[pathIdx].lines[line].controlPoints[point - 1]) {
              additionalPaths[pathIdx].lines[line].controlPoints[point - 1].x = inchX;
              additionalPaths[pathIdx].lines[line].controlPoints[point - 1].y = inchY;
            }
            additionalPaths = [...additionalPaths];
          }
          // Auto-save changes to additional path files
          saveAdditionalPath(pathIdx).catch(err => console.error('Failed to auto-save additional path:', err));
        } else {
          // Handle path point dragging
          const line = Number(currentElem.split("-")[1]) - 1;
          const point = Number(currentElem.split("-")[2]);

          if (line === -1) {
            // This is the starting point
            if (startPoint.locked) return;
            startPoint.x = inchX;
            startPoint.y = inchY;
          } else if (lines[line]) {
            if (point === 0 && lines[line].endPoint) {
              lines[line].endPoint.x = inchX;
              lines[line].endPoint.y = inchY;
            } else {
              if (lines[line]?.locked) return;
              lines[line].controlPoints[point - 1].x = inchX;
              lines[line].controlPoints[point - 1].y = inchY;
            }
          }
        }
      } else {
        if (
          (elem?.id.startsWith("point") && !isLockedPathElem(elem.id)) ||
          elem?.id.startsWith("second-point") ||
          elem?.id.startsWith("additional-path-") ||
          elem?.id.startsWith("obstacle")
        ) {
          two.renderer.domElement.style.cursor = "pointer";
          currentElem = elem.id;
        } else {
          two.renderer.domElement.style.cursor = "auto";
          currentElem = null;
        }
      }
    });

    two.renderer.domElement.addEventListener("mousedown", (evt: MouseEvent) => {
      if (simulatorMode) return;

      if (currentElem && isLockedPathElem(currentElem)) {
        isDown = false;
        return;
      }

      isDown = true;

      if (currentElem) {
        const rect = two.renderer.domElement.getBoundingClientRect();
        const mouseX = x.invert(evt.clientX - rect.left);
        const mouseY = y.invert(evt.clientY - rect.top);

        let objectX = 0;
        let objectY = 0;

        if (currentElem.startsWith("obstacle-")) {
          const parts = currentElem.split("-");
          const shapeIdx = Number(parts[1]);
          const vertexIdx = Number(parts[2]);
          if (shapes[shapeIdx]?.vertices[vertexIdx]) {
            objectX = shapes[shapeIdx].vertices[vertexIdx].x;
            objectY = shapes[shapeIdx].vertices[vertexIdx].y;
          }
        } else if (currentElem.startsWith("second-point-")) {
          const parts = currentElem.split("-");
          const line = Number(parts[2]) - 1;
          const point = Number(parts[3]);

          if (line === -1) {
            if (secondStartPoint) {
              objectX = secondStartPoint.x;
              objectY = secondStartPoint.y;
            }
          } else if (secondLines[line]) {
            if (point === 0 && secondLines[line].endPoint) {
              objectX = secondLines[line].endPoint.x;
              objectY = secondLines[line].endPoint.y;
            } else if (secondLines[line].controlPoints[point - 1]) {
              objectX = secondLines[line].controlPoints[point - 1].x;
              objectY = secondLines[line].controlPoints[point - 1].y;
            }
          }
        } else if (currentElem.startsWith("additional-path-")) {
          const parts = currentElem.split("-");
          const pathIdx = Number(parts[2]);
          const line = Number(parts[4]) - 1;
          const point = Number(parts[5]);

          if (additionalPaths[pathIdx]) {
            if (line === -1) {
              // Starting point
              if (additionalPaths[pathIdx].startPoint) {
                objectX = additionalPaths[pathIdx].startPoint.x;
                objectY = additionalPaths[pathIdx].startPoint.y;
              }
            } else if (additionalPaths[pathIdx].lines[line]) {
              if (point === 0 && additionalPaths[pathIdx].lines[line].endPoint) {
                objectX = additionalPaths[pathIdx].lines[line].endPoint.x;
                objectY = additionalPaths[pathIdx].lines[line].endPoint.y;
              } else if (additionalPaths[pathIdx].lines[line].controlPoints[point - 1]) {
                objectX = additionalPaths[pathIdx].lines[line].controlPoints[point - 1].x;
                objectY = additionalPaths[pathIdx].lines[line].controlPoints[point - 1].y;
              }
            }
          }
        } else {
          const line = Number(currentElem.split("-")[1]) - 1;
          const point = Number(currentElem.split("-")[2]);

          if (line === -1) {
            objectX = startPoint.x;
            objectY = startPoint.y;
          } else if (lines[line]) {
            if (point === 0 && lines[line].endPoint) {
              objectX = lines[line].endPoint.x;
              objectY = lines[line].endPoint.y;
            } else if (lines[line].controlPoints[point - 1]) {
              objectX = lines[line].controlPoints[point - 1].x;
              objectY = lines[line].controlPoints[point - 1].y;
            }
          }
        }

        dragOffset = {
          x: objectX - mouseX,
          y: objectY - mouseY,
        };
      }
    });

    two.renderer.domElement.addEventListener("mouseup", () => {
      if (simulatorMode) return;

      isDown = false;
      dragOffset = { x: 0, y: 0 };
      recordChange();
    });

    // Double-click on the field to create a new path at that position
    two.renderer.domElement.addEventListener("dblclick", (evt: MouseEvent) => {
      if (simulatorMode) return;

      // Ignore dblclicks on existing points/lines
      const elem = document.elementFromPoint(evt.clientX, evt.clientY);
      if (
        elem?.id &&
        (elem.id.startsWith("point") ||
          elem.id.startsWith("obstacle") ||
          elem.id.startsWith("line"))
      ) {
        return;
      }

      const rect = two.renderer.domElement.getBoundingClientRect();
      const rawInchX = x.invert(evt.clientX - rect.left);
      const rawInchY = y.invert(evt.clientY - rect.top);

      // Apply grid snapping if enabled
      const currentGridSize = $gridSize;
      const currentSnapToGrid = $snapToGrid;
      const currentShowGrid = $showGrid;

      let inchX = rawInchX;
      let inchY = rawInchY;

      if (currentSnapToGrid && currentShowGrid && currentGridSize > 0) {
        inchX = Math.round(rawInchX / currentGridSize) * currentGridSize;
        inchY = Math.round(rawInchY / currentGridSize) * currentGridSize;
      }

      // Clamp to field boundaries
      inchX = Math.max(0, Math.min(FIELD_SIZE, inchX));
      inchY = Math.max(0, Math.min(FIELD_SIZE, inchY));

      // Create a new line with endPoint at the clicked position
      const newLine: Line = {
        id: `line-${Math.random().toString(36).slice(2)}`,
        endPoint: {
          x: inchX,
          y: inchY,
          heading: "tangential",
          reverse: false,
        },
        controlPoints: [],
        color: getRandomColor(),
        locked: false,
        waitBeforeMs: 0,
        waitAfterMs: 0,
        waitBeforeName: "",
        waitAfterName: "",
      };

      lines = [...lines, newLine];
      sequence = [...sequence, { kind: "path", lineId: newLine.id! }];
      recordChange();
      two.update();
    });
  });
  document.addEventListener("keydown", function (evt) {
    if (simulatorMode) return;
    if (evt.code === "Space" && document.activeElement === document.body) {
      if (playing) {
        pause();
      } else {
        play();
      }
    }
  });
  async function saveFile() {
    try {
      const content = JSON.stringify(
        {
          startPoint,
          lines,
          shapes,
          sequence,
          settings,
          version: "1.2.1",
          timestamp: new Date().toISOString(),
        },
        null,
        2,
      );

      if ($currentFilePath) {
        await browserFileStore.writeFile($currentFilePath, content);
        isUnsaved.set(false);
        // Provide simple feedback
        alert(`Saved to project storage: ${$currentFilePath}`);
      } else {
        // No current project file selected — save into browser cache as a new file
        const defaultName = `path_${Date.now()}.pp`;
        await browserFileStore.writeFile(defaultName, content);
        currentFilePath.set(defaultName);
        isUnsaved.set(false);
        alert(`Saved to project storage as: ${defaultName}`);
      }
    } catch (err) {
      console.error("Failed to save project to storage:", err);
      alert("Failed to save project to browser storage.");
    }
  }

  async function loadFile(evt: Event) {
    const elem = evt.target as HTMLInputElement;
    const file = elem.files?.[0];

    if (!file) return;

    // Check if file is a .pp file
    if (!file.name.endsWith(".pp")) {
      alert("Please select a .pp file");
      // Reset the file input
      elem.value = "";
      return;
    }

    // Parse and load the uploaded file, then cache it into the browser store.
    loadTrajectoryFromFile(evt, async (data) => {
      // Ensure startPoint has all required fields
      startPoint = data.startPoint || {
        x: 72,
        y: 72,
        heading: "tangential",
        reverse: false,
      };

      // Normalize lines with all required fields
      const normalizedLines = normalizeLines(data.lines || []);
      lines = normalizedLines;

      // Derive sequence from data or create default
      sequence = (
        data.sequence && data.sequence.length
          ? data.sequence
          : normalizedLines.map((ln) => ({
              kind: "path",
              lineId: ln.id!,
            }))
      ) as SequenceItem[];

      // Load shapes with defaults
      shapes = data.shapes || [];

      // Load settings (including robot size) if present
      if (data.settings) {
        settings = { ...settings, ...data.settings };
        robotWidth = settings.rWidth;
        robotHeight = settings.rHeight;
      }

      isUnsaved.set(false);
      recordChange();

      // Cache the uploaded file into the browser-backed store for later access
      try {
        const content = JSON.stringify(data);
        await browserFileStore.writeFile(file.name, content);
        currentFilePath.set(file.name);
      } catch (err) {
        console.warn("Failed to cache uploaded file to store:", err);
      }
    });

    // Reset the file input
    elem.value = "";
  }

  // Electron file-copying logic removed — browser store and upload are used instead.

  // Helper function to load data into app state
  function loadData(data: any) {
    // Ensure startPoint has all required fields
    startPoint = data.startPoint || {
      x: 72,
      y: 72,
      heading: "tangential",
      reverse: false,
    };

    // Normalize lines with all required fields
    const normalizedLines = normalizeLines(data.lines || []);
    lines = normalizedLines;

    // Derive sequence from data or create default
    sequence = (
      data.sequence && data.sequence.length
        ? data.sequence
        : normalizedLines.map((ln) => ({
            kind: "path",
            lineId: ln.id!,
          }))
    ) as SequenceItem[];

    // Load shapes with defaults
    shapes = data.shapes || [];

    // Load settings (including robot size) if present
    if (data.settings) {
      settings = { ...settings, ...data.settings };
      robotWidth = settings.rWidth;
      robotHeight = settings.rHeight;
    }

    isUnsaved.set(false);
    recordChange();
  }

  function toHeadingDegrees(point: Point, position: "start" | "end"): number {
    if (!point) return 0;
    if (point.heading === "linear") {
      return position === "start" ? (point.startDeg ?? 0) : (point.endDeg ?? 0);
    }
    if (point.heading === "constant") {
      return point.degrees ?? 0;
    }
    return 0;
  }

  function buildOptimizationPayload(lineIndex: number) {
    const line = lines[lineIndex];
    if (!line) throw new Error("Line not found");

    const startPt =
      lineIndex === 0 ? startPoint : lines[lineIndex - 1]?.endPoint;
    if (!startPt) throw new Error("Missing start point for optimization");

    const waypoints = [startPt, ...line.controlPoints, line.endPoint].map(
      (p) => [p.x, p.y],
    );

    return {
      waypoints,
      start_heading_degrees: toHeadingDegrees(startPt, "start"),
      end_heading_degrees: toHeadingDegrees(line.endPoint, "end"),
      x_velocity: settings.xVelocity,
      y_velocity: settings.yVelocity,
      angular_velocity: settings.aVelocity,
      friction_coefficient: settings.kFriction,
      robot_width: settings.rWidth,
      robot_height: settings.rHeight,
      min_coord_field: 0,
      max_coord_field: FIELD_SIZE,
      interpolation:
        line.endPoint.heading === "tangential"
          ? "tangent"
          : line.endPoint.heading === "constant"
            ? "constant"
            : "linear",
      obstacles: shapes.map((shape) => shape.vertices.map((v) => [v.x, v.y])),
    };
  }

  function sleep(ms: number) {
    return new Promise((res) => setTimeout(res, ms));
  }

  async function runOptimization(payload: any) {
    const response = await fetch(`${OPTIMIZER_BASE_URL}`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(payload),
    });

    if (!response.ok) {
      const errorText = await response.text().catch(() => "");
      throw new Error(
        `Optimizer request failed (${response.status}): ${errorText || response.statusText}`,
      );
    }

    const data = await response.json();
    if (data?.status === "completed" && data.result) {
      return data.result;
    }
    if (data?.status === "error") {
      throw new Error(
        `Optimization failed: ${data.message || "Unknown error"}`,
      );
    }
    throw new Error("Unexpected API response format");
  }

  async function optimizeLine(
    lineId: string,
    targetControlPointIndex?: number,
  ) {
    const lineIndex = lines.findIndex((l) => l.id === lineId);
    if (lineIndex === -1) {
      alert("Could not find line to optimize.");
      return;
    }

    if (optimizingLineIds[lineId]) return;
    optimizingLineIds = { ...optimizingLineIds, [lineId]: true };

    try {
      const payload = buildOptimizationPayload(lineIndex);
      const result = await runOptimization(payload);

      const optimizedWaypoints = Array.isArray(result?.optimized_waypoints)
        ? result.optimized_waypoints
        : Array.isArray(result)
          ? result
          : null;

      if (!optimizedWaypoints || optimizedWaypoints.length < 2) {
        throw new Error("Unexpected optimizer response format.");
      }

      const interior = optimizedWaypoints
        .slice(1, optimizedWaypoints.length - 1)
        .map((p: number[]) => ({ x: p[0], y: p[1] }));

      const newLines = [...lines];
      const current = newLines[lineIndex];

      if (typeof targetControlPointIndex === "number") {
        // Only replace the targeted control point; keep others and endpoint untouched
        const replacement =
          interior[targetControlPointIndex] ?? interior[interior.length - 1];
        if (replacement) {
          const cps = [...current.controlPoints];
          if (cps[targetControlPointIndex]) {
            cps[targetControlPointIndex] = replacement;
            newLines[lineIndex] = {
              ...current,
              controlPoints: cps,
            };
            lines = normalizeLines(newLines);
            recordChange();
          }
        }
      } else {
        // Replace entire line (control points and endpoint)
        newLines[lineIndex] = {
          ...current,
          endPoint: {
            ...current.endPoint,
            x: optimizedWaypoints[optimizedWaypoints.length - 1][0],
            y: optimizedWaypoints[optimizedWaypoints.length - 1][1],
          },
          controlPoints: interior,
        };
        lines = normalizeLines(newLines);
        recordChange();
      }
    } catch (err) {
      console.error(err);
      alert((err as Error).message || "Optimization failed.");
    } finally {
      optimizingLineIds = { ...optimizingLineIds, [lineId]: false };
    }
  }

  async function optimizeAllLines() {
    if (optimizingAll) return;
    optimizingAll = true;
    try {
      for (const ln of lines) {
        if (!ln?.id) continue;
        await optimizeLine(ln.id);
      }
    } finally {
      optimizingAll = false;
    }
  }

  function loadRobot(evt: Event) {
    loadRobotImage(evt, () => updateRobotImageDisplay());
  }

  function addNewLine() {
    lines = [
      ...lines,
      {
        id: `line-${Math.random().toString(36).slice(2)}`,
        endPoint: {
          x: _.random(36, 108),
          y: _.random(36, 108),
          heading: "tangential",
          reverse: true,
        } as Point,
        controlPoints: [],
        color: getRandomColor(),
        locked: false,
        waitBeforeMs: 0,
        waitAfterMs: 0,
        waitBeforeName: "",
        waitAfterName: "",
      },
    ];
    sequence = [
      ...sequence,
      { kind: "path", lineId: lines[lines.length - 1].id! },
    ];
    recordChange();
  }

  function addControlPoint() {
    if (lines.length > 0) {
      const lastLine = lines[lines.length - 1];
      lastLine.controlPoints.push({
        x: _.random(36, 108),
        y: _.random(36, 108),
      });
      recordChange();
    }
  }

  function removeControlPoint() {
    if (lines.length > 0) {
      const lastLine = lines[lines.length - 1];
      if (lastLine.controlPoints.length > 0) {
        lastLine.controlPoints.pop();
        recordChange();
      }
    }
  }

  // Keyboard shortcuts for quick path editing
  hotkeys("w", function (event, handler) {
    if (simulatorMode) return;
    event.preventDefault();
    addNewLine();
  });
  hotkeys("a", function (event, handler) {
    if (simulatorMode) return;
    event.preventDefault();
    addControlPoint();
    two.update();
  });
  hotkeys("s", function (event, handler) {
    if (simulatorMode) return;
    event.preventDefault();
    removeControlPoint();
    two.update();
  });
  hotkeys("cmd+z, ctrl+z", function (event) {
    if (simulatorMode) return;
    event.preventDefault();
    undoAction();
  });
  hotkeys("cmd+shift+z, ctrl+shift+z, ctrl+y", function (event) {
    if (simulatorMode) return;
    event.preventDefault();
    redoAction();
  });
  function applyTheme(theme: "light" | "dark" | "auto") {
    let actualTheme = theme;
    if (theme === "auto") {
      // Check system preference
      if (
        window.matchMedia &&
        window.matchMedia("(prefers-color-scheme: dark)").matches
      ) {
        actualTheme = "dark";
      } else {
        actualTheme = "light";
      }
    }

    if (actualTheme === "dark") {
      document.documentElement.classList.add("dark");
    } else {
      document.documentElement.classList.remove("dark");
    }
  }

  // Watch for theme changes in settings
  $: if (settings) {
    applyTheme(settings.theme);
  }

  // Watch for system theme changes if auto mode is enabled
  let mediaQuery: MediaQueryList;
  onMount(() => {
    if (settings?.theme === "auto") {
      mediaQuery = window.matchMedia("(prefers-color-scheme: dark)");
      const handleSystemThemeChange = () => {
        if (settings.theme === "auto") {
          applyTheme("auto");
        }
      };
      mediaQuery.addEventListener("change", handleSystemThemeChange);

      return () => {
        mediaQuery.removeEventListener("change", handleSystemThemeChange);
      };
    }
  });

  // Auto-export for CI/testing: if the app is loaded with URL hash #export-gif-test, automatically run GIF export once mounted
  onMount(() => {
    if (
      typeof window !== "undefined" &&
      window.location &&
      window.location.hash === "#export-gif-test"
    ) {
      // Delay slightly to allow initial rendering and Two.js to initialize
      setTimeout(async () => {
        try {
          // auto GIF export removed (exportGif deleted)
          console.log("Auto GIF export skipped (exportGif removed)");
        } catch (err) {
          console.error("Auto GIF export failed:", err);
        }
      }, 1500);
    }

    // Handle save dialog event
    const handleSaveDialog = async (event: any) => {
      const { fileName } = event.detail;
      isSaving = true;
      try {
        // Create a full file path with .pp extension if not present
        const fullFileName = fileName.endsWith(".pp") ? fileName : fileName + ".pp";
        
        // Call the file manager's save function through the browser file store
        const fileData = JSON.stringify({
          startPoint,
          lines,
          shapes,
          sequence,
          settings,
        });
        
        await browserFileStore.writeFile(fullFileName, fileData);
        currentFilePath.set(fullFileName);
        isUnsaved.set(false);
        
        // Show success feedback
        showSaveDialog = false;
      } catch (error) {
        console.error("Save failed:", error);
        alert("Failed to save file: " + (error instanceof Error ? error.message : String(error)));
      } finally {
        isSaving = false;
      }
    };

    window.addEventListener("save", handleSaveDialog);

    // Handle dual path save dialog event
    const handleDualPathSave = async (event: any) => {
      const { target } = event.detail;
      isSaving = true;
      try {
        if (target === "first" && $currentFilePath) {
          const fileData = JSON.stringify({
            startPoint,
            lines,
            shapes,
            sequence,
            settings,
            version: "1.2.1",
            timestamp: new Date().toISOString(),
          });
          await browserFileStore.writeFile($currentFilePath, fileData);
          isUnsaved.set(false);
        } else if (target === "second" && $secondFilePath) {
          const fileData = JSON.stringify({
            startPoint: secondStartPoint,
            lines: secondLines,
            shapes: secondShapes,
            sequence: secondSequence,
            settings,
            version: "1.2.1",
            timestamp: new Date().toISOString(),
          });
          await browserFileStore.writeFile($secondFilePath, fileData);
        } else if (target === "both") {
          // Save first path
          if ($currentFilePath) {
            const fileData1 = JSON.stringify({
              startPoint,
              lines,
              shapes,
              sequence,
              settings,
              version: "1.2.1",
              timestamp: new Date().toISOString(),
            });
            await browserFileStore.writeFile($currentFilePath, fileData1);
          }
          // Save second path
          if ($secondFilePath) {
            const fileData2 = JSON.stringify({
              startPoint: secondStartPoint,
              lines: secondLines,
              shapes: secondShapes,
              sequence: secondSequence,
              settings,
              version: "1.2.1",
              timestamp: new Date().toISOString(),
            });
            await browserFileStore.writeFile($secondFilePath, fileData2);
          }
          isUnsaved.set(false);
        }
        showDualPathSaveDialog = false;
      } catch (error) {
        console.error("Dual path save failed:", error);
        alert("Failed to save: " + (error instanceof Error ? error.message : String(error)));
      } finally {
        isSaving = false;
      }
    };

    window.addEventListener("saveDualPath", handleDualPathSave);

    return () => {
      window.removeEventListener("save", handleSaveDialog);
      window.removeEventListener("saveDualPath", handleDualPathSave);
    };
  });
</script>

<svelte:window
  on:mousemove={(e) => {
    // (debug window dragging removed)
  }}
  on:mouseup={() => {}}
/>

{#if !simulatorMode}
  <Navbar
    bind:lines
    bind:startPoint
    bind:shapes
    bind:sequence
    bind:secondStartPoint
    bind:secondLines
    bind:secondShapes
    bind:secondSequence
    bind:settings
    bind:robotWidth
    bind:robotHeight
    {percent}
    {saveProject}
    {saveFileAs}
    {loadFile}
    {undoAction}
    {redoAction}
    {recordChange}
    {canUndo}
    {canRedo}
    {optimizeAllLines}
    {optimizingAll}
    {twoElement}
    bind:playing
    {play}
    {pause}
  />
{/if}

<SaveDialog
  bind:isOpen={showSaveDialog}
  bind:isSaving
  fileName={$currentFilePath?.split(/[\\/]/).pop()?.replace(/\.pp$/, "") || "my_path"}
/>

<DualPathSaveDialog bind:isOpen={showDualPathSaveDialog} />

<!--   {saveFile} -->
<div
  class={`w-screen h-screen p-2 flex flex-row justify-center items-center gap-2 ${simulatorMode ? "pt-2" : "pt-20"}`}
>
  {#if simulatorMode}
    <div class="h-full w-[360px] hidden xl:flex items-center justify-end pr-1">
      <div class="w-[340px] rounded-2xl border border-cyan-300/35 bg-slate-950/80 text-slate-100 p-4 shadow-[0_14px_34px_rgba(0,0,0,0.45)] backdrop-blur-sm">
        <div class="text-lg font-bold tracking-wide text-cyan-100">FTC Decode Driving Sim</div>
        <div class="mt-2 text-sm leading-6 text-slate-200">
          <div>Drive: Left stick (field-centric)</div>
          <div>Turn: Right stick X-axis</div>
          <div>Intake: RB/RT | Hold Shoot: A | Reset: Y</div>
          <div>Keyboard P1: WASD move | J/L turn | LShift intake | \ shoot</div>
          <div>Intake is rear edge only (touch required)</div>
          <div>Lever dump: hit lever from side edge</div>
        </div>
        <div class="mt-3 rounded-xl border border-cyan-200/35 bg-slate-900/80 p-3 shadow-inner">
          <div class="text-[11px] uppercase tracking-[0.18em] text-cyan-200/75">Driver Period</div>
          <div class="mt-1 text-4xl font-black tabular-nums text-cyan-100">{simMatchClockText}</div>
          <div class={`mt-1 text-xs font-semibold ${simMatchActive ? "text-cyan-200/80" : "text-amber-300"}`}>
            {simMatchActive ? "LIVE" : "MATCH ENDED"}
          </div>
        </div>
        <div class="mt-3 grid grid-cols-2 gap-2 text-sm">
          <div class="rounded-lg bg-slate-900/65 border border-slate-700/80 px-2 py-1.5">Held: {simHeldBalls} total</div>
          <div class="rounded-lg border border-cyan-300/55 bg-cyan-900/35 px-2 py-1.5 text-xl font-extrabold text-cyan-100">Score: {simScore}</div>
          <div class="rounded-lg bg-fuchsia-900/35 border border-fuchsia-400/50 px-2 py-1.5">Purple Scored: {simColorScores.purple}</div>
          <div class="rounded-lg bg-emerald-900/35 border border-emerald-400/50 px-2 py-1.5">Green Scored: {simColorScores.green}</div>
          <div class="rounded-lg bg-slate-900/65 border border-slate-700/80 px-2 py-1.5 col-span-2 truncate">{simControllerSummary}</div>
          <label class="rounded-lg bg-slate-900/65 border border-slate-700/80 px-2 py-1.5 col-span-2 flex items-center justify-between gap-2">
            <span class="text-xs text-slate-300">P1 Controller</span>
            <select
              class="bg-slate-800 border border-slate-600 rounded px-2 py-1 text-xs max-w-[210px]"
              value={simControllerSelections[0]}
              on:change={(e) => handleControllerSelectionChange(0, e)}
            >
              <option value={simAutoControllerSelection}>Auto</option>
              <option value={simKeyboardControllerSelection}>Keyboard</option>
              {#each simConnectedControllers as controller}
                <option value={controller.index}>{controller.index}: {controller.name}</option>
              {/each}
            </select>
          </label>
          <label class="rounded-lg bg-slate-900/65 border border-slate-700/80 px-2 py-1.5 col-span-2 flex items-center justify-between gap-2">
            <span class="text-xs text-slate-300">P2 Controller</span>
            <select
              class="bg-slate-800 border border-slate-600 rounded px-2 py-1 text-xs max-w-[210px]"
              value={simControllerSelections[1]}
              on:change={(e) => handleControllerSelectionChange(1, e)}
            >
              <option value={simAutoControllerSelection}>Auto</option>
              {#each simConnectedControllers as controller}
                <option value={controller.index}>{controller.index}: {controller.name}</option>
              {/each}
            </select>
          </label>
          <div class="rounded-lg bg-slate-900/65 border border-slate-700/80 px-2 py-1.5 col-span-2">Balls: 16 purple, 8 green</div>
        </div>
      </div>
    </div>
  {/if}

  <div class="flex h-full justify-center items-center flex-1 min-w-0">
    <div
      bind:this={twoElement}
      bind:clientWidth={width}
      bind:clientHeight={height}
      class="h-full aspect-square rounded-lg shadow-md bg-neutral-50 dark:bg-neutral-900 relative overflow-clip"
      role="application"
      style="
    user-select: none;
    -webkit-user-select: none;
    -moz-user-select: none;
    -ms-user-select: none;
    -webkit-touch-callout: none;
    -webkit-tap-highlight-color: transparent;
    user-drag: none;
    -webkit-user-drag: none;
    -khtml-user-drag: none;
    -moz-user-drag: none;
    -ms-user-drag: none;
    -o-user-drag: none;
  "
      on:contextmenu={(e) => e.preventDefault()}
      on:dragstart={(e) => e.preventDefault()}
      on:selectstart={(e) => e.preventDefault()}
      tabindex="-1"
    >
      <img
        src={settings.fieldMap
          ? `/fields/${settings.fieldMap}`
          : "/fields/decode.webp"}
        alt="Field"
        class="absolute top-0 left-0 w-full h-full rounded-lg z-10"
        style="
    background: transparent; 
    pointer-events: none; 
    user-select: none; 
    -webkit-user-select: none;
    -moz-user-select: none;
    -ms-user-select: none;
    -webkit-touch-callout: none;
    -webkit-tap-highlight-color: transparent;
    user-drag: none;
    -webkit-user-drag: none;
    -moz-user-drag: none;
    -ms-user-drag: none;
    -o-user-drag: none;
  "
        draggable="false"
        on:error={(e) => {
          console.error("Failed to load field map:", settings.fieldMap);
          e.target.src = "/fields/decode.webp"; // Fallback
        }}
        on:dragstart={(e) => e.preventDefault()}
        on:selectstart={(e) => e.preventDefault()}
      />
      {#if !simulatorMode}
        <MathTools {x} {y} {twoElement} {robotXY} {robotHeading} />
      {/if}

      {#if simulatorMode}
        {#each simRobots as simRobot, simRobotIdx}
          {@const intakeHeadingRad = (simRobot.heading * Math.PI) / 180}
          {@const intakeCenterX = simRobot.xy.x - Math.cos(intakeHeadingRad) * x(robotHeight) / 2}
          {@const intakeCenterY = simRobot.xy.y - Math.sin(intakeHeadingRad) * x(robotHeight) / 2}
          {@const intakeDirX = Math.cos(intakeHeadingRad + Math.PI / 2)}
          {@const intakeDirY = Math.sin(intakeHeadingRad + Math.PI / 2)}
          {@const intakeHalfWidth = x(robotWidth) / 2}

          <div
            class={`absolute z-30 rounded-full ${simRobotIdx === 0 ? "bg-amber-300/90" : "bg-sky-300/90"}`}
            style={`left:${intakeCenterX - intakeDirX * intakeHalfWidth}px; top:${intakeCenterY - intakeDirY * intakeHalfWidth}px; width:${x(robotWidth)}px; height:3px; transform-origin: 0 50%; transform: rotate(${(intakeHeadingRad + Math.PI / 2) * 180 / Math.PI}deg);`}
            title={`Robot ${simRobotIdx + 1} Rear intake edge`}
          />

          <div
            class={`absolute z-30 rounded-full border-2 ${simRobotIdx === 0 ? "border-amber-300/90 bg-amber-400/60" : "border-sky-300/90 bg-sky-400/60"}`}
            style={`left:${simRobot.xy.x - 8}px; top:${simRobot.xy.y - 8}px; width:16px; height:16px;`}
            title={`Robot ${simRobotIdx + 1} Turret`}
          />

          <div
            class={`absolute z-30 rounded-sm ${simRobotIdx === 0 ? "bg-amber-200/90" : "bg-sky-200/90"}`}
            style={`left:${simRobot.xy.x}px; top:${simRobot.xy.y - 2}px; width:${Math.max(20, x(robotWidth) * 0.7)}px; height:4px; transform-origin: 0 50%; transform: translateY(-50%) rotate(${(simRobot.turretAngleRad * 180) / Math.PI}deg);`}
            title={`Robot ${simRobotIdx + 1} Turret barrel`}
          />

          {#each simRobot.heldBallColors as color, heldIdx}
            <div
              class={`absolute z-30 rounded-full border ${color === "purple" ? "bg-fuchsia-500 border-fuchsia-300" : "bg-emerald-500 border-emerald-300"}`}
              style={`left:${simRobot.xy.x - 12 + heldIdx * 16}px; top:${simRobot.xy.y - x(robotHeight) / 2 - 18}px; width:16px; height:16px;`}
              title={`Robot ${simRobotIdx + 1} Held ${color} ball`}
            />
          {/each}
        {/each}

        {#each simGoals as goal}
          <div
            class="absolute z-20 rounded-full border-2 border-cyan-300/90 bg-cyan-500/30"
            style={`left:${goal.x - goal.r}px; top:${goal.y - goal.r}px; width:${goal.r * 2}px; height:${goal.r * 2}px;`}
            title={goal.label}
          />
        {/each}

        {#each simClassifiers as classifier}
          <div
            class="absolute rounded-md border-2 border-slate-100/95 bg-slate-950/82"
            style={`left:${classifier.x}px; top:${classifier.y}px; width:${classifier.width}px; height:${classifier.height}px; z-index: 90;`}
            title="Classifier"
          />

          <div
            class="absolute rounded-sm bg-slate-100/25 border-2 border-cyan-100/85"
            style={`left:${classifier.x + 3}px; top:${classifier.y + 3}px; width:${classifier.width - 6}px; height:${classifier.height - 6}px; z-index: 95;`}
            title="Inventory tube"
          />

          <div
            class="absolute text-[10px] font-bold text-slate-100"
            style={`left:${classifier.x + 4}px; top:${classifier.y + 4}px; z-index: 96;`}
          >
            {simGoalInventories[classifier.goalId]?.length || 0}/9
          </div>
        {/each}

        {#each simLevers as lever}
          {@const leverOpen = isLeverOpen(lever)}
          <div
            class={`absolute z-20 rounded-md border-2 ${leverOpen ? "border-emerald-200/90 bg-emerald-600/50" : "border-red-200/90 bg-red-600/50"}`}
            style={`left:${lever.x - lever.width / 2}px; top:${lever.y - lever.height / 2}px; width:${lever.width}px; height:${lever.height}px;`}
            title={lever.label}
          >
            <div class={`absolute inset-0 flex items-center justify-center text-[9px] font-semibold ${leverOpen ? "text-emerald-100" : "text-red-100"}`}>L</div>
          </div>
        {/each}

        {#each simBalls.filter((ball) => !ball.collected && !ball.scored) as ball}
          <div
            class={`absolute z-20 rounded-full border ${ball.color === "purple" ? "bg-fuchsia-500 border-fuchsia-300" : "bg-emerald-500 border-emerald-300"}`}
            style={`left:${ball.x - ball.r}px; top:${ball.y - ball.r}px; width:${ball.r * 2}px; height:${ball.r * 2}px;`}
            title="Ball"
          />
        {/each}

        {#each simTransferBalls as transfer}
          <div
            class={`absolute rounded-full border-2 ${transfer.color === "purple" ? "bg-fuchsia-400 border-fuchsia-100" : "bg-emerald-400 border-emerald-100"}`}
            style={`left:${transfer.x - transfer.r}px; top:${transfer.y - transfer.r}px; width:${transfer.r * 2}px; height:${transfer.r * 2}px; z-index:140; box-shadow: 0 0 10px rgba(255,255,255,0.6);`}
            title="Ball transferring to inventory"
          />
        {/each}

        {#each simProjectiles as projectile}
          {@const projectileState = getProjectileRenderState(projectile)}
          <div
            class={`absolute z-40 rounded-full border-2 ${projectile.color === "purple" ? "bg-fuchsia-400 border-fuchsia-200" : "bg-emerald-400 border-emerald-200"}`}
            style={`left:${projectileState.x - simBallRadiusPx}px; top:${projectileState.y - simBallRadiusPx}px; width:${simBallRadiusPx * 2}px; height:${simBallRadiusPx * 2}px;`}
            title="Shot ball"
          />
        {/each}

      {/if}
      <!-- Main robot: only show in normal mode -->
      {#if $activePaths.length === 0}
        {#if simulatorMode}
          {#each simRobots as simRobot, simRobotIdx}
            <img
              src={settings.robotImage || "/robot.png"}
              alt={`Sim Robot ${simRobotIdx + 1}`}
              style={`position: absolute; top: ${simRobot.xy.y}px;
left: ${simRobot.xy.x}px; transform: translate(-50%, -50%) rotate(${simRobot.heading}deg); z-index: ${22 - simRobotIdx}; width: ${x(robotWidth)}px; height: ${x(robotHeight)}px;user-select: none; -webkit-user-select: none; -moz-user-select: none;-ms-user-select: none;
pointer-events: none; opacity: ${simRobotIdx === 0 ? 1 : 0.88};`}
              draggable="false"
              on:error={(e) => {
                console.error("Failed to load robot image:", settings.robotImage);
                e.target.src = "/robot.png";
              }}
              on:dragstart={(e) => e.preventDefault()}
              on:selectstart={(e) => e.preventDefault()}
            />
          {/each}
        {:else}
          <img
            src={settings.robotImage || "/robot.png"}
            alt="Robot"
            style={`position: absolute; top: ${robotXY.y}px;
left: ${robotXY.x}px; transform: translate(-50%, -50%) rotate(${robotHeading}deg); z-index: 20; width: ${x(robotWidth)}px; height: ${x(robotHeight)}px;user-select: none; -webkit-user-select: none; -moz-user-select: none;-ms-user-select: none;
pointer-events: none;`}
            draggable="false"
            on:error={(e) => {
              console.error("Failed to load robot image:", settings.robotImage);
              e.target.src = "/robot.png"; // Fallback to default
            }}
            on:dragstart={(e) => e.preventDefault()}
            on:selectstart={(e) => e.preventDefault()}
          />
          <!-- Heading arrow for main robot -->
          {#if settings.showHeadingArrow}
            <svg
              style={`position: absolute; top: ${robotXY.y}px; left: ${robotXY.x}px; z-index: 21; pointer-events: none; overflow: visible;`}
              width="1"
              height="1"
            >
              <defs>
                <marker
                  id="arrowhead-main"
                  markerWidth="10"
                  markerHeight="10"
                  refX="6.5"
                  refY="3"
                  orient="auto"
                >
                  <polygon
                    points="0 0, 7 3, 0 6"
                    fill={settings.headingArrowColor || "#ffffff"}
                  />
                </marker>
              </defs>
              <line
                x1="0"
                y1="0"
                x2="{(settings.headingArrowLength || 50) * Math.cos(-robotHeading * Math.PI / 180)}"
                y2="{(settings.headingArrowLength || 50) * -Math.sin(-robotHeading * Math.PI / 180)}"
                stroke={settings.headingArrowColor || "#ffffff"}
                stroke-width={settings.headingArrowThickness || 3}
                marker-end="url(#arrowhead-main)"
              />
            </svg>
          {/if}
        {/if}
      {/if}
      <!-- Second robot: only show in dual path mode (not multi-path mode) -->
      {#if $activePaths.length === 0 && $dualPathMode}
        <img
          src={settings.robotImage || "/robot.png"}
          alt="Robot 2"
          style={`position: absolute; top: ${secondRobotXY.y}px;
left: ${secondRobotXY.x}px; transform: translate(-50%, -50%) rotate(${secondRobotHeading}deg); z-index: 19; width: ${x(robotWidth)}px; height: ${x(robotHeight)}px;user-select: none; -webkit-user-select: none; -moz-user-select: none;-ms-user-select: none;
pointer-events: none; opacity: 0.8;`}
          draggable="false"
          on:error={(e) => {
            console.error("Failed to load robot image:", settings.robotImage);
            e.target.src = "/robot.png";
          }}
          on:dragstart={(e) => e.preventDefault()}
          on:selectstart={(e) => e.preventDefault()}
        />
        <!-- Heading arrow for second robot -->
        {#if settings.showHeadingArrow}
          <svg
            style={`position: absolute; top: ${secondRobotXY.y}px; left: ${secondRobotXY.x}px; z-index: 19; pointer-events: none; overflow: visible; opacity: 0.8;`}
            width="1"
            height="1"
          >
            <defs>
              <marker
                id="arrowhead-second"
                markerWidth="10"
                markerHeight="10"
                refX="6.5"
                refY="3"
                orient="auto"
              >
                <polygon
                  points="0 0, 7 3, 0 6"
                  fill={settings.headingArrowColor || "#ffffff"}
                />
              </marker>
            </defs>
            <line
              x1="0"
              y1="0"
              x2="{(settings.headingArrowLength || 50) * Math.cos(-secondRobotHeading * Math.PI / 180)}"
              y2="{(settings.headingArrowLength || 50) * -Math.sin(-secondRobotHeading * Math.PI / 180)}"
              stroke={settings.headingArrowColor || "#ffffff"}
              stroke-width={settings.headingArrowThickness || 3}
              marker-end="url(#arrowhead-second)"
            />
          </svg>
        {/if}
      {/if}
      <!-- Additional robots: only show in multi-path mode -->
      {#if $activePaths.length > 0}
        {#each additionalRobotStates as robotState, idx}
          <img
            src={settings.robotImage || "/robot.png"}
            alt="Robot {idx + 1}"
            style={`position: absolute; top: ${robotState.xy.y}px;
left: ${robotState.xy.x}px; transform: translate(-50%, -50%) rotate(${robotState.heading}deg); z-index: ${20 - idx}; width: ${x(robotWidth)}px; height: ${x(robotHeight)}px;user-select: none; -webkit-user-select: none; -moz-user-select: none;-ms-user-select: none;
pointer-events: none; opacity: ${1.0 - idx * 0.15};`}
            draggable="false"
            on:error={(e) => {
              console.error("Failed to load robot image:", settings.robotImage);
              e.target.src = "/robot.png";
            }}
            on:dragstart={(e) => e.preventDefault()}
            on:selectstart={(e) => e.preventDefault()}
          />
          <!-- Heading arrow for additional robots -->
          {#if settings.showHeadingArrow}
            <svg
              style={`position: absolute; top: ${robotState.xy.y}px; left: ${robotState.xy.x}px; z-index: ${20 - idx}; pointer-events: none; overflow: visible; opacity: ${1.0 - idx * 0.15};`}
              width="1"
              height="1"
            >
              <defs>
                <marker
                  id="arrowhead-{idx}"
                  markerWidth="10"
                  markerHeight="10"
                  refX="6.5"
                  refY="3"
                  orient="auto"
                >
                  <polygon
                    points="0 0, 7 3, 0 6"
                    fill={settings.headingArrowColor || "#ffffff"}
                  />
                </marker>
              </defs>
              <line
                x1="0"
                y1="0"
                x2="{(settings.headingArrowLength || 50) * Math.cos(-robotState.heading * Math.PI / 180)}"
                y2="{(settings.headingArrowLength || 50) * -Math.sin(-robotState.heading * Math.PI / 180)}"
                stroke={settings.headingArrowColor || "#ffffff"}
                stroke-width={settings.headingArrowThickness || 3}
                marker-end="url(#arrowhead-{idx})"
              />
            </svg>
          {/if}
        {/each}
      {/if}
    </div>
  </div>
  {#if simulatorMode}
    <div class="h-full w-[360px] hidden xl:flex items-center justify-start pl-1">
      <div class="w-[350px] rounded-2xl border border-amber-300/40 bg-slate-950/82 text-slate-100 p-4 shadow-[0_14px_34px_rgba(0,0,0,0.45)] backdrop-blur-sm">
        <div class="text-lg font-bold tracking-wide text-amber-100 mb-3">Drive Tuning</div>
        <label class="flex items-center justify-between gap-3 mb-2 text-sm">
          <span class="text-slate-200">Top speed</span>
          <input class="w-28 rounded-lg bg-slate-900 border border-slate-500 px-2 py-1.5 text-right" type="number" min="20" max="900" step="5" bind:value={simMoveSpeedPxPerSec} />
        </label>
        <label class="flex items-center justify-between gap-3 mb-2 text-sm">
          <span class="text-slate-200">Accel</span>
          <input class="w-28 rounded-lg bg-slate-900 border border-slate-500 px-2 py-1.5 text-right" type="number" min="50" max="5000" step="10" bind:value={simAccel} />
        </label>
        <label class="flex items-center justify-between gap-3 mb-2 text-sm">
          <span class="text-slate-200">Decel</span>
          <input class="w-28 rounded-lg bg-slate-900 border border-slate-500 px-2 py-1.5 text-right" type="number" min="50" max="5000" step="10" bind:value={simDecel} />
        </label>
        <label class="flex items-center justify-between gap-3 mb-2 text-sm">
          <span class="text-slate-200">Turn speed</span>
          <input class="w-28 rounded-lg bg-slate-900 border border-slate-500 px-2 py-1.5 text-right" type="number" min="20" max="720" step="5" bind:value={simTurnSpeedDegPerSec} />
        </label>
        <label class="flex items-center justify-between gap-3 mb-2 text-sm">
          <span class="text-slate-200">Turn accel</span>
          <input class="w-28 rounded-lg bg-slate-900 border border-slate-500 px-2 py-1.5 text-right" type="number" min="50" max="5000" step="10" bind:value={simRotAccel} />
        </label>
        <label class="flex items-center justify-between gap-3 text-sm">
          <span class="text-slate-200">Turn decel</span>
          <input class="w-28 rounded-lg bg-slate-900 border border-slate-500 px-2 py-1.5 text-right" type="number" min="50" max="5000" step="10" bind:value={simRotDecel} />
        </label>
      </div>
    </div>
  {/if}
  {#if !simulatorMode}
    <ControlTab
      bind:playing
      {play}
      {pause}
      bind:startPoint
      bind:lines
      bind:sequence
      bind:robotWidth
      bind:robotHeight
      bind:settings
      bind:percent
      bind:robotXY
      bind:robotHeading
      bind:shapes
      {x}
      {y}
      {animationDuration}
      {handleSeek}
      bind:loopAnimation
      {resetAnimation}
      {recordChange}
      {optimizeLine}
      {optimizingLineIds}
      {optimizeAllLines}
      {optimizingAll}
    />
  {/if}
</div>
