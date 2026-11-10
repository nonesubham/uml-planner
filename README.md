# UML Planner

A minimalist UML class diagram editor built with Vue 3 and Vue Flow. Create class diagrams with draggable nodes, connectable relationship edges, and a clean monochrome theme.

## Features

- **Class nodes** — Add UML classes with attributes and methods, each with access modifiers (+, -, #)
- **Smart placement** — New nodes appear at canvas center; overlapping nodes are offset automatically
- **Drag & drop** — Freely reposition nodes on an infinite canvas
- **Relationship edges** — Drag from any handle to connect classes; choose from 5 relationship types (association, inheritance, composition, aggregation, dependency)
- **Edit nodes** — Double-click any node to modify its name, attributes, or methods; drag hamburger handles to reorder attributes and methods
- **Edit edges** — Double-click any edge to change its relationship type
- **Delete nodes** — Remove unwanted classes with the × button on the node header
- **Session persistence** — Diagram is auto-saved to localStorage and restored on revisit
- **Save/Open** — Export diagrams as `.umld` files and re-import them later
- **Export PNG/JPG/SVG** — Save your diagram as a raster image (PNG/JPG) or vector graphic (SVG) with authentic bezier curves and UML relationship markers
- **Zoom & pan** — Controls panel for zoom in/out and fit-to-view
- **Monochrome theme** — Pure black canvas with high-contrast white-on-dark styling

## Tech Stack

| Tool | Purpose |
|---|---|
| [Vue 3](https://vuejs.org/) | UI framework |
| [Vue Flow](https://vueflow.dev/) | Interactive node graph engine |
| [Vite](https://vite.dev/) | Build tool |

## Getting Started

```bash
npm install
npm run dev
```

Open [http://localhost:5173](http://localhost:5173).

## Build

```bash
npm run build
npm run preview
```

## Usage

1. Click **+ Class** to open the class editor
2. Fill in the class name, access modifier, attributes, and methods
3. Click **Add Class** to place the node on the canvas
4. **Drag** nodes to reposition them
5. **Drag from a handle** (small circles on node edges) to another node to create a relationship
6. **Choose a relation type** from the popup: *has a*, *is a*, *contains*, *uses*
7. **Double-click** a node to edit its fields
8. **Save** — Download diagram as `.umld` file
9. **Open** — Import a `.umld` file to restore a saved diagram
10. **PNG / JPG / SVG** — Export your diagram as a PNG, JPG, or SVG vector file

## Project Structure

```
src/
├── components/
│   ├── UMLDesigner.vue    # Main editor (canvas, toolbar, modals, export, graph logic)
│   ├── UMLClassNode.vue   # Custom class card node component with handles
│   ├── UMLClassEdge.vue   # Custom relationship edge with direction-aware labels
│   └── UMLGroupNode.vue   # Visual boundary container for grouped nodes
├── assets/                # Static assets
├── main.js                # Entry point
├── style.css              # Tailwind import
└── App.vue                # Root component
```

## Why Vue, Not React?

This tool could've been built with React — but I don't know React, and I didn't want to React to the idea of using React to build a tool in React. I like my frameworks like I like my error messages: predictable and not written by Meta. Vue just worked without needing 47 hooks, a corporate manifesto, or a RFC to change a prop name. Sometimes the right framework is the one that lets you ship instead of letting you refactor.

## License

MIT
