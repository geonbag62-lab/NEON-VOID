# NEON//VOID

> A high-performance procedural cyberpunk roguelite built entirely with Python and Pygame.

![Python](https://img.shields.io/badge/Python-3.10%2B-3776AB?style=for-the-badge&logo=python)
![Pygame](https://img.shields.io/badge/Pygame-2.5%2B-00A86B?style=for-the-badge)
![License](https://img.shields.io/badge/License-MIT-black?style=for-the-badge)

## Overview

NEON//VOID is a top-down action roguelite focused on fast combat, procedural enemy scaling, upgrade choices and multi-phase boss encounters.

The project is intentionally implemented without a game engine.

Everything is handled directly through Python and Pygame:

- Rendering
- Input
- Collision
- Enemy AI
- Projectile simulation
- Particle effects
- Camera movement
- Screen shake
- Procedural enemy spawning
- Upgrade system
- XP progression
- Boss state machine
- Persistent high score

## Features

### Combat

- Mouse-aimed plasma weapon
- Variable projectile speed
- Projectile piercing
- Enemy projectiles
- Collision detection
- Damage numbers
- Hit particles
- Screen shake
- Dash invulnerability

### Player progression

Players gain XP by defeating enemies.

Every level grants a randomized selection of three upgrades.

Available upgrades include:

- VOID CORE
- OVERDRIVE
- RAPID FIRE
- PLASMA
- PHASE
- PIERCER
- VITALITY
- MAGNET

The resulting build changes every run.

### Enemy AI

Enemies dynamically react to the player's position.

Different behaviors include:

- Pursuit
- Strafing
- Ranged attacks
- Elite variants
- Scaling health
- Scaling damage
- Scaling movement speed

### Boss system

Every fifth room introduces the VOID WARDEN.

The boss contains three combat phases.

```text
PHASE 1
  Targeted projectile attacks

PHASE 2
  Radial projectile patterns

PHASE 3
  High-speed bullet patterns
  Increased attack frequency
