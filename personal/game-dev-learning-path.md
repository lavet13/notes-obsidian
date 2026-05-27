---
id: game-dev-learning-path
aliases:
  - game-dev-learning-path
tags: []
---

# Beginner Game Development Learning Path
From TypeScript/Neovim Background to Simple 3D Game

Target: Build a small 3D game (e.g. character movement, collectibles, basic enemies) while understanding the fundamentals.

---

## Phase 0: Mindset & Setup (1-2 days)

### Goals
- Understand the difference between game engines and writing from scratch
- Set up tools that respect your Neovim workflow

### Steps

1. Download Godot 4.4+ (Official Editor)
   - Go to: [godotengine](https://godotengine.org)
   - Download the standard version (not the C# version for now)

2. Configure Godot for Neovim
   - Open Godot Editor
   - Go to: Editor → Editor Settings → Text Editor → External
   - Enable "Use External Editor"
   - Set Exec Path to your Neovim executable
   - Set Exec Flags for Neovim (check Godot documentation if needed)

3. Neovim LSP Support (Recommended)
   - Install gdscript language server via your plugin manager (e.g. mason.nvim)

4. Create Project Folder
   mkdir -p ~/projects/megabonk-clone
   cd ~/projects/megabonk-clone

---

## Phase 1: Godot Fundamentals (Week 1)

Focus: Learn Godot while writing real code in Neovim.

### Daily Plan

Day 1-2
- Complete the official Godot tutorial: "Your First 3D Game" (very important)

Day 3-4
- Study:
  - Nodes & Scene system
  - GDScript syntax (similar to TypeScript + Python)
  - Input handling, CharacterBody3D, Camera3D, basic movement & jumping

Day 5-7
- Build a small playable prototype:
  - Player that can move (WASD) and jump
  - Several collectible items
  - Simple score UI
  - Basic environment and lighting

---

## Phase 2: Intermediate Concepts (Week 2)

- State machines for player and enemies
- Basic enemy AI (move toward player)
- Particles, sound effects, animations
- Scene management and signals
- Introduction to Godot's MultiplayerAPI

---

## Phase 3: Decision Point (End of Week 3)

### Option A: Stay with Godot (Recommended)
- Continue building your full small game
- Learn exporting to Desktop and Web
- Great productivity + understanding balance

### Option B: Move to raylib (Deeper Understanding)
- Install Clang compiler
- Download raylib
- Port your prototype to C using Neovim

---

## Recommended Project Structure

megabonk-clone/
├── main.tscn
├── scenes/
│   ├── player.tscn
│   ├── collectible.tscn
│   └── enemy.tscn
├── scripts/
│   ├── player.gd
│   ├── game_manager.gd
│   └── enemy.gd
├── assets/
├── autoload/
└── .git/

---

## Tops for Your Background

- GDScript will feel familiar coming from TypeScript.
- Use Godot Editor for visual work (scenes, nodes), Neovim for scripting.
- Free assets: [kenney.nl](https://kenney.nl)

Start Today: Open Godot and begin the official "Your First 3D Game" tutorial.

Once you finish Phase 1, reply and I can give you more detailed code examples or expand the guide.

[chatting with grok](https://grok.com/c/39f303ed-bb3f-48e9-8f36-34fcda080d3e?rid=35b0e310-e015-4e0d-8e1c-3759210a637a)
