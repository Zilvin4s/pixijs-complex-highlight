<template>
  <canvas id="canvas"></canvas>
</template>

<script setup>
import * as PIXI from "pixi.js";
import ClipperLib from "clipper-lib";

const app = new PIXI.Application({ background: "#000", resizeTo: window });

document.body.appendChild(app.view);

// Function to create a square
function createSquare(x, y, size, color = 0xde3249) {
  const square = new PIXI.Graphics();
  square.beginFill(color);
  square.drawRect(0, 0, size, size);
  square.endFill();
  square.x = x;
  square.y = y;
  square.size = size;
  square.interactive = true;
  square.buttonMode = true;

  // Store the square's bounds for collision detection
  square.bounds = new PIXI.Rectangle(square.x, square.y, size, size);

  // Add the square to the stage
  app.stage.addChild(square);

  return square;
}

// Array to store all squares
const squares = [];

// Create multiple squares at random positions
for (let i = 0; i < 100; i++) {
  const size = 50;
  const x = Math.random() * (app.screen.width - size);
  const y = Math.random() * (app.screen.height - size);
  const square = createSquare(x, y, size);
  squares.push(square);
}

// Function to detect connected squares using Depth-First Search (DFS)
function findConnectedSquares(clickedSquare, squares) {
  const connected = [];
  const visited = new Set();

  function dfs(square) {
    connected.push(square);
    visited.add(square);

    for (const otherSquare of squares) {
      if (!visited.has(otherSquare) && areSquaresConnected(square, otherSquare)) {
        dfs(otherSquare);
      }
    }
  }

  dfs(clickedSquare);
  return connected;
}

// Function to check if two squares are connected (touching or overlapping)
function areSquaresConnected(squareA, squareB) {
  return squareA.bounds.intersects(squareB.bounds);
}

// Function to draw the highlight around connected squares
function drawHighlight(squares) {
  // Remove existing highlight if any
  if (app.highlight) {
    app.stage.removeChild(app.highlight);
    app.highlight.destroy();
    app.highlight = null;
  }

  // Build the combined outline of connected squares
  const outlinePaths = calculateCombinedOutline(squares);

  // Offset the outline by {number} of pixels using Clipper.js
  const offsetPaths = offsetPolygon(outlinePaths, 5);

  // Draw the polygon highlight
  const highlight = new PIXI.Graphics();
  highlight.lineStyle(5, 0xffffff); // 5px {color} border
  highlight.beginFill(0xff0000, 0); // Transparent fill

  // Draw each offset path
  for (const path of offsetPaths) {
    const points = path.map((p) => new PIXI.Point(p.X, p.Y));
    highlight.drawPolygon(points);
  }

  highlight.endFill();

  // Add the highlight to the stage
  app.stage.addChild(highlight);
  app.highlight = highlight;
}

// Function to calculate the combined outline of the connected squares
function calculateCombinedOutline(squares) {
  // Create a Clipper Paths array
  const paths = [];

  for (const square of squares) {
    const x = square.x;
    const y = square.y;
    const size = square.size;

    // Define the square as a path
    const squarePath = [
      { X: x, Y: y },
      { X: x + size, Y: y },
      { X: x + size, Y: y + size },
      { X: x, Y: y + size },
    ];

    paths.push(squarePath);
  }

  // Union the squares to get the combined outline
  const cpr = new ClipperLib.Clipper();
  const solution = new ClipperLib.Paths();

  cpr.AddPaths(paths, ClipperLib.PolyType.ptSubject, true);
  cpr.Execute(
    ClipperLib.ClipType.ctUnion,
    solution,
    ClipperLib.PolyFillType.pftNonZero,
    ClipperLib.PolyFillType.pftNonZero
  );

  return solution;
}

// Function to offset the polygon path inward by a given distance using Clipper.js
function offsetPolygon(paths, offset) {
  const miterLimit = 2;
  const arcTolerance = 0.25;

  const co = new ClipperLib.ClipperOffset(miterLimit, arcTolerance);
  co.AddPaths(paths, ClipperLib.JoinType.jtMiter, ClipperLib.EndType.etClosedPolygon);

  const offsetPaths = new ClipperLib.Paths();
  co.Execute(offsetPaths, offset);

  return offsetPaths;
}

// Add event listeners to squares
for (const square of squares) {
  square.on("pointerdown", function () {
    // Find all connected squares
    const connectedSquares = findConnectedSquares(this, squares);

    // Draw the highlight around them
    drawHighlight(connectedSquares);
  });
}
</script>
