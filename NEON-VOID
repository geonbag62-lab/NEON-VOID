# main.py
# NEON//VOID — single-file Pygame roguelite
# Python 3.10+ / pygame 2.5+

import pygame
import random
import math
import json
import os
from dataclasses import dataclass, field

pygame.init()
pygame.mixer.init()

WIDTH, HEIGHT = 1280, 720
FPS = 120
TITLE = "NEON//VOID"

screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption(TITLE)
clock = pygame.time.Clock()

FONT = pygame.font.SysFont("consolas", 18)
SMALL = pygame.font.SysFont("consolas", 14)
BIG = pygame.font.SysFont("consolas", 34, bold=True)
HUGE = pygame.font.SysFont("consolas", 72, bold=True)

SAVE_FILE = "save.json"

# ---------------------------------------------------------------------------
# Utilities
# ---------------------------------------------------------------------------

def clamp(v, a, b):
    return max(a, min(b, v))


def lerp(a, b, t):
    return a + (b - a) * t


def vec_from_angle(angle):
    return pygame.Vector2(math.cos(angle), math.sin(angle))


def angle_to(a, b):
    return math.atan2(b.y - a.y, b.x - a.x)


def draw_text(surface, text, pos, font=FONT, color=(230, 240, 255), center=False):
    img = font.render(str(text), True, color)
    rect = img.get_rect()
    if center:
        rect.center = pos
    else:
        rect.topleft = pos
    surface.blit(img, rect)


def circle_surface(radius, color, alpha=255):
    s = pygame.Surface((radius * 2 + 4, radius * 2 + 4), pygame.SRCALPHA)
    pygame.draw.circle(s, (*color, alpha), (radius + 2, radius + 2), radius)
    return s


# ---------------------------------------------------------------------------
# Camera
# ---------------------------------------------------------------------------

class Camera:
    def __init__(self):
        self.pos = pygame.Vector2()
        self.shake = 0.0

    def update(self, target, dt):
        self.pos += (target - self.pos) * min(1.0, dt * 8.0)
        if self.shake > 0:
            self.shake = max(0, self.shake - dt)

    def offset(self):
        if self.shake <= 0:
            return pygame.Vector2()

        return pygame.Vector2(
            random.uniform(-self.shake, self.shake),
            random.uniform(-self.shake, self.shake)
        )


# ---------------------------------------------------------------------------
# Particles
# ---------------------------------------------------------------------------

@dataclass
class Particle:
    pos: pygame.Vector2
    vel: pygame.Vector2
    life: float
    size: float
    color: tuple
    gravity: float = 0
    shrink: float = 0.8

    def update(self, dt):
        self.pos += self.vel * dt
        self.vel.y += self.gravity * dt
        self.life -= dt
        self.size *= max(0, 1 - self.shrink * dt)

    def draw(self, surface, camera):
        if self.life <= 0 or self.size <= 0.5:
            return

        p = self.pos - camera.pos + pygame.Vector2(WIDTH / 2, HEIGHT / 2)
        r = max(1, int(self.size))
        pygame.draw.circle(surface, self.color, p, r)


class ParticleSystem:
    def __init__(self):
        self.items = []

    def burst(self, pos, color, amount=20, speed=180):
        for _ in range(amount):
            a = random.random() * math.tau
            v = vec_from_angle(a) * random.uniform(speed * .2, speed)
            self.items.append(
                Particle(
                    pygame.Vector2(pos),
                    v,
                    random.uniform(.25, .8),
                    random.uniform(2, 6),
                    color
                )
            )

    def trail(self, pos, color):
        if random.random() < .65:
            self.items.append(
                Particle(
                    pygame.Vector2(pos),
                    pygame.Vector2(random.uniform(-15, 15), random.uniform(-15, 15)),
                    .22,
                    random.uniform(1, 3),
                    color
                )
            )

    def update(self, dt):
        for p in self.items:
            p.update(dt)
        self.items = [p for p in self.items if p.life > 0 and p.size > .5]

    def draw(self, surface, camera):
        for p in self.items:
            p.draw(surface, camera)


# ---------------------------------------------------------------------------
# Floating damage / text
# ---------------------------------------------------------------------------

@dataclass
class FloatingText:
    pos: pygame.Vector2
    text: str
    color: tuple
    life: float = .8

    def update(self, dt):
        self.pos.y -= 35 * dt
        self.life -= dt

    def draw(self, surface, camera):
        if self.life <= 0:
            return

        p = self.pos - camera.pos + pygame.Vector2(WIDTH / 2, HEIGHT / 2)
        draw_text(surface, self.text, p, SMALL, self.color, True)


# ---------------------------------------------------------------------------
# Projectile
# ---------------------------------------------------------------------------

@dataclass
class Projectile:
    pos: pygame.Vector2
    vel: pygame.Vector2
    damage: float
    radius: float
    color: tuple
    life: float = 2.0
    pierce: int = 0
    enemy: bool = False
    hit_ids: set = field(default_factory=set)

    def update(self, dt):
        self.pos += self.vel * dt
        self.life -= dt

    def draw(self, surface, camera):
        p = self.pos - camera.pos + pygame.Vector2(WIDTH / 2, HEIGHT / 2)

        glow = pygame.Surface((50, 50), pygame.SRCALPHA)
        pygame.draw.circle(glow, (*self.color, 35), (25, 25), 20)
        pygame.draw.circle(glow, (*self.color, 90), (25, 25), 10)
        pygame.draw.circle(glow, (*self.color, 255), (25, 25), int(self.radius))
        surface.blit(glow, glow.get_rect(center=p))


# ---------------------------------------------------------------------------
# Upgrade
# ---------------------------------------------------------------------------

UPGRADES = [
    ("VOID CORE", "Maximum HP +25", "max_hp", 25),
    ("OVERDRIVE", "Move speed +15%", "speed", .15),
    ("RAPID FIRE", "Fire rate +18%", "fire_rate", .18),
    ("PLASMA", "Damage +25%", "damage", .25),
    ("PHASE", "Dash cooldown -25%", "dash", .25),
    ("PIERCER", "Projectiles pierce +1 enemy", "pierce", 1),
    ("VITALITY", "Heal 20 HP", "heal", 20),
    ("MAGNET", "XP pickup range +45%", "magnet", .45),
]


# ---------------------------------------------------------------------------
# Player
# ---------------------------------------------------------------------------

class Player:
    def __init__(self, game):
        self.game = game
        self.pos = pygame.Vector2(0, 0)
        self.vel = pygame.Vector2()

        self.radius = 16

        self.max_hp = 100
        self.hp = 100

        self.speed = 300
        self.damage = 24
        self.fire_rate = .22
        self.fire_timer = 0

        self.dash_cooldown = 1.0
        self.dash_timer = 0
        self.dash_time = 0
        self.dash_velocity = pygame.Vector2()

        self.xp = 0
        self.level = 1
        self.xp_next = 100

        self.pierce = 0
        self.magnet = 110

        self.invincible = 0

    def gain_xp(self, amount):
        self.xp += amount

        if self.xp >= self.xp_next:
            self.xp -= self.xp_next
            self.level += 1
            self.xp_next = int(self.xp_next * 1.35)
            self.game.state = "UPGRADE"
            self.game.generate_upgrades()

    def take_damage(self, amount):
        if self.invincible > 0 or self.dash_time > 0:
            return

        self.hp -= amount
        self.invincible = .45
        self.game.camera.shake = 10
        self.game.particles.burst(self.pos, (255, 60, 90), 15, 150)

        if self.hp <= 0:
            self.hp = 0
            self.game.state = "GAMEOVER"

    def dash(self):
        if self.dash_timer > 0 or self.dash_time > 0:
            return

        keys = pygame.key.get_pressed()
        d = pygame.Vector2(
            keys[pygame.K_d] - keys[pygame.K_a],
            keys[pygame.K_s] - keys[pygame.K_w]
        )

        if d.length_squared() == 0:
            mouse = pygame.Vector2(pygame.mouse.get_pos())
            center = pygame.Vector2(WIDTH / 2, HEIGHT / 2)
            d = mouse - center

        if d.length_squared() == 0:
            return

        d = d.normalize()
        self.dash_velocity = d * 1050
        self.dash_time = .13
        self.dash_timer = self.dash_cooldown
        self.invincible = .25

    def shoot(self, target):
        if self.fire_timer > 0:
            return

        direction = target - self.pos
        if direction.length_squared() == 0:
            return

        direction = direction.normalize()

        self.game.projectiles.append(
            Projectile(
                self.pos + direction * 22,
                direction * 850,
                self.damage,
                5,
                (80, 230, 255),
                1.5,
                self.pierce
            )
        )

        self.fire_timer = self.fire_rate
        self.game.particles.trail(self.pos, (80, 230, 255))

    def update(self, dt):
        self.fire_timer = max(0, self.fire_timer - dt)
        self.dash_timer = max(0, self.dash_timer - dt)
        self.invincible = max(0, self.invincible - dt)

        if self.dash_time > 0:
            self.dash_time -= dt
            self.pos += self.dash_velocity * dt
            self.game.particles.trail(self.pos, (100, 220, 255))
            return

        keys = pygame.key.get_pressed()
        move = pygame.Vector2(
            keys[pygame.K_d] - keys[pygame.K_a],
            keys[pygame.K_s] - keys[pygame.K_w]
        )

        if move.length_squared():
            move = move.normalize()

        self.vel = move * self.speed
        self.pos += self.vel * dt

        mouse_world = (
            pygame.Vector2(pygame.mouse.get_pos())
            - pygame.Vector2(WIDTH / 2, HEIGHT / 2)
            + self.game.camera.pos
        )

        if pygame.mouse.get_pressed()[0]:
            self.shoot(mouse_world)

        self.pos.x = clamp(self.pos.x, -1150, 1150)
        self.pos.y = clamp(self.pos.y, -600, 600)

    def draw(self, surface, camera):
        p = self.pos - camera.pos + pygame.Vector2(WIDTH / 2, HEIGHT / 2)

        if self.invincible > 0 and int(self.invincible * 20) % 2 == 0:
            return

        direction = pygame.Vector2(pygame.mouse.get_pos()) - p
        if direction.length_squared() == 0:
            direction = pygame.Vector2(1, 0)
        direction = direction.normalize()

        side = pygame.Vector2(-direction.y, direction.x)

        points = [
            p + direction * 22,
            p - direction * 12 + side * 13,
            p - direction * 5,
            p - direction * 12 - side * 13
        ]

        glow = pygame.Surface((90, 90), pygame.SRCALPHA)
        pygame.draw.circle(glow, (50, 210, 255, 30), (45, 45), 38)
        surface.blit(glow, glow.get_rect(center=p))

        pygame.draw.polygon(surface, (60, 220, 255), points)
        pygame.draw.polygon(surface, (220, 250, 255), points, 2)


# ---------------------------------------------------------------------------
# Enemy
# ---------------------------------------------------------------------------

class Enemy:
    def __init__(self, game, pos, elite=False):
        self.game = game
        self.pos = pygame.Vector2(pos)

        self.elite = elite
        self.radius = 18 if not elite else 26

        scale = 1 + game.wave * .07

        self.max_hp = (55 if not elite else 220) * scale
        self.hp = self.max_hp

        self.speed = (105 if not elite else 75) * (1 + game.wave * .015)
        self.damage = (12 if not elite else 25) * scale

        self.attack_timer = random.uniform(.5, 1.5)
        self.strafe = random.choice([-1, 1])
        self.id = id(self)

    def update(self, dt):
        player = self.game.player

        d = player.pos - self.pos
        dist = d.length()

        if dist > 1:
            direction = d.normalize()

            if dist > 230:
                self.pos += direction * self.speed * dt
            else:
                tangent = pygame.Vector2(-direction.y, direction.x)
                self.pos += (direction * .15 + tangent * self.strafe * .5) * self.speed * dt

        self.attack_timer -= dt

        if self.attack_timer <= 0 and dist < 500:
            self.attack_timer = random.uniform(1.1, 2.0)

            if dist > 80:
                direction = d.normalize()
                self.game.enemy_projectiles.append(
                    Projectile(
                        self.pos,
                        direction * (280 if not self.elite else 360),
                        self.damage,
                        6 if not self.elite else 9,
                        (255, 90, 120),
                        3,
                        enemy=True
                    )
                )

        if dist < self.radius + player.radius:
            player.take_damage(self.damage * dt)

    def hit(self, damage, projectile):
        self.hp -= damage

        self.game.floating.append(
            FloatingText(
                self.pos - pygame.Vector2(0, self.radius),
                str(int(damage)),
                (255, 220, 130)
            )
        )

        self.game.particles.burst(self.pos, (255, 100, 130), 5, 90)

        if self.hp <= 0:
            self.game.kill_enemy(self)

    def draw(self, surface, camera):
        p = self.pos - camera.pos + pygame.Vector2(WIDTH / 2, HEIGHT / 2)

        color = (255, 65, 105) if not self.elite else (190, 80, 255)

        pygame.draw.circle(surface, (20, 10, 25), p, self.radius + 6)
        pygame.draw.circle(surface, color, p, self.radius)
        pygame.draw.circle(surface, (255, 220, 230), p, max(2, self.radius // 4))

        # HP bar
        w = self.radius * 2.4
        ratio = clamp(self.hp / self.max_hp, 0, 1)
        pygame.draw.rect(
            surface,
            (30, 30, 40),
            (p.x - w / 2, p.y - self.radius - 12, w, 4)
        )
        pygame.draw.rect(
            surface,
            color,
            (p.x - w / 2, p.y - self.radius - 12, w * ratio, 4)
        )


# ---------------------------------------------------------------------------
# Boss
# ---------------------------------------------------------------------------

class Boss:
    def __init__(self, game):
        self.game = game
        self.pos = pygame.Vector2(0, -350)

        self.max_hp = 4200 + game.wave * 500
        self.hp = self.max_hp

        self.radius = 72
        self.phase = 1
        self.timer = 0
        self.attack_timer = 2
        self.teleport_timer = 7

        self.angle = 0

    def update(self, dt):
        player = self.game.player
        self.timer += dt
        self.attack_timer -= dt
        self.teleport_timer -= dt

        ratio = self.hp / self.max_hp

        if ratio <= .66:
            self.phase = 2
        if ratio <= .33:
            self.phase = 3

        target = player.pos

        direction = target - self.pos

        if direction.length_squared():
            direction = direction.normalize()

        self.pos += direction * (35 + self.phase * 10) * dt

        if self.attack_timer <= 0:
            self.attack_timer = max(.25, 1.15 - self.phase * .18)
            self.attack()

        if self.teleport_timer <= 0:
            self.teleport_timer = max(2.5, 7 - self.phase)
            self.pos = player.pos + vec_from_angle(random.random() * math.tau) * 280
            self.game.particles.burst(self.pos, (190, 80, 255), 45, 250)

    def attack(self):
        player = self.game.player

        if self.phase == 1:
            direction = angle_to(self.pos, player.pos)

            for spread in (-.15, 0, .15):
                d = vec_from_angle(direction + spread)
                self.game.enemy_projectiles.append(
                    Projectile(
                        self.pos,
                        d * 360,
                        18,
                        9,
                        (255, 70, 150),
                        4,
                        enemy=True
                    )
                )

        elif self.phase == 2:
            for i in range(12):
                a = self.angle + math.tau * i / 12
                self.game.enemy_projectiles.append(
                    Projectile(
                        self.pos,
                        vec_from_angle(a) * 300,
                        22,
                        8,
                        (190, 80, 255),
                        4,
                        enemy=True
                    )
                )
            self.angle += .18

        else:
            direction = angle_to(self.pos, player.pos)

            for i in range(18):
                a = direction + math.tau * i / 18
                self.game.enemy_projectiles.append(
                    Projectile(
                        self.pos,
                        vec_from_angle(a) * 420,
                        28,
                        7,
                        (255, 50, 100),
                        3,
                        enemy=True
                    )
                )

            self.game.camera.shake = 8

    def hit(self, damage):
        self.hp -= damage

        self.game.floating.append(
            FloatingText(
                self.pos - pygame.Vector2(0, 85),
                str(int(damage)),
                (255, 220, 100)
            )
        )

        self.game.particles.burst(self.pos, (200, 80, 255), 3, 100)

        if self.hp <= 0:
            self.hp = 0
            self.game.boss_defeated()

    def draw(self, surface, camera):
        p = self.pos - camera.pos + pygame.Vector2(WIDTH / 2, HEIGHT / 2)

        glow = pygame.Surface((240, 240), pygame.SRCALPHA)

        pygame.draw.circle(glow, (180, 60, 255, 30), (120, 120), 110)
        pygame.draw.circle(glow, (255, 50, 120, 35), (120, 120), 80)

        surface.blit(glow, glow.get_rect(center=p))

        pygame.draw.circle(surface, (15, 8, 30), p, self.radius + 8)
        pygame.draw.circle(surface, (130, 45, 210), p, self.radius)
        pygame.draw.circle(surface, (30, 10, 50), p, self.radius - 18)

        for i in range(4):
            a = self.timer * (1 if i % 2 else -1) + i * math.pi / 2
            q = p + vec_from_angle(a) * 54
            pygame.draw.circle(surface, (255, 90, 180), q, 7)

        pygame.draw.circle(surface, (255, 220, 250), p, 13)

        w = 650
        x = WIDTH / 2 - w / 2

        pygame.draw.rect(surface, (20, 15, 30), (x, 25, w, 14))
        pygame.draw.rect(
            surface,
            (220, 60, 150),
            (x, 25, w * max(0, self.hp / self.max_hp), 14)
        )

        draw_text(
            surface,
            f"VOID WARDEN // PHASE {self.phase}",
            (WIDTH / 2, 5),
            SMALL,
            (255, 180, 240),
            True
        )


# ---------------------------------------------------------------------------
# Game
# ---------------------------------------------------------------------------

class Game:
    def __init__(self):
        self.state = "MENU"

        self.player = Player(self)
        self.camera = Camera()
        self.particles = ParticleSystem()

        self.projectiles = []
        self.enemy_projectiles = []
        self.enemies = []
        self.floating = []

        self.wave = 1
        self.room = 1
        self.score = 0
        self.kills = 0

        self.room_timer = 0
        self.spawn_timer = 0

        self.upgrade_choices = []
        self.boss = None

        self.high_score = self.load_save()

    # ---------------------------- persistence ----------------------------

    def load_save(self):
        if not os.path.exists(SAVE_FILE):
            return 0

        try:
            with open(SAVE_FILE, "r", encoding="utf-8") as f:
                return int(json.load(f).get("high_score", 0))
        except Exception:
            return 0

    def save(self):
        try:
            with open(SAVE_FILE, "w", encoding="utf-8") as f:
                json.dump({"high_score": self.high_score}, f, indent=2)
        except Exception:
            pass

    # ---------------------------- game flow ----------------------------

    def start(self):
        self.player = Player(self)
        self.camera = Camera()
        self.particles = ParticleSystem()

        self.projectiles.clear()
        self.enemy_projectiles.clear()
        self.enemies.clear()
        self.floating.clear()

        self.wave = 1
        self.room = 1
        self.score = 0
        self.kills = 0

        self.room_timer = 0
        self.spawn_timer = 0
        self.boss = None

        self.state = "PLAYING"

    def next_room(self):
        self.room += 1
        self.wave += 1
        self.room_timer = 0
        self.spawn_timer = 0

        self.player.hp = min(self.player.max_hp, self.player.hp + 15)

        self.enemies.clear()
        self.projectiles.clear()
        self.enemy_projectiles.clear()

        if self.room % 5 == 0:
            self.boss = Boss(self)
            self.state = "BOSS"
        else:
            self.boss = None
            self.state = "PLAYING"

    def generate_upgrades(self):
        self.upgrade_choices = random.sample(UPGRADES, 3)

    def apply_upgrade(self, index):
        name, desc, stat, value = self.upgrade_choices[index]

        if stat == "max_hp":
            self.player.max_hp += value
            self.player.hp += value

        elif stat == "speed":
            self.player.speed *= 1 + value

        elif stat == "fire_rate":
            self.player.fire_rate *= 1 - value

        elif stat == "damage":
            self.player.damage *= 1 + value

        elif stat == "dash":
            self.player.dash_cooldown *= 1 - value

        elif stat == "pierce":
            self.player.pierce += value

        elif stat == "heal":
            self.player.hp = min(self.player.max_hp, self.player.hp + value)

        elif stat == "magnet":
            self.player.magnet *= 1 + value

        self.state = "PLAYING"

    def kill_enemy(self, enemy):
        if enemy not in self.enemies:
            return

        self.enemies.remove(enemy)

        xp = 30 if not enemy.elite else 100

        self.player.gain_xp(xp)

        self.score += 100 if not enemy.elite else 500
        self.kills += 1

        self.particles.burst(
            enemy.pos,
            (255, 80, 140) if not enemy.elite else (190, 80, 255),
            24,
            220
        )

    def boss_defeated(self):
        self.score += 10000
        self.kills += 1
        self.particles.burst(self.boss.pos, (255, 80, 200), 100, 400)
        self.boss = None
        self.next_room()

    # ---------------------------- update ----------------------------

    def spawn_enemy(self):
        if len(self.enemies) >= min(35, 8 + self.wave * 2):
            return

        edge = random.choice([0, 1, 2, 3])

        if edge == 0:
            pos = pygame.Vector2(random.uniform(-1100, 1100), -570)
        elif edge == 1:
            pos = pygame.Vector2(random.uniform(-1100, 1100), 570)
        elif edge == 2:
            pos = pygame.Vector2(-1120, random.uniform(-550, 550))
        else:
            pos = pygame.Vector2(1120, random.uniform(-550, 550))

        elite = random.random() < min(.28, self.wave * .025)

        self.enemies.append(Enemy(self, pos, elite))

    def update(self, dt):
        self.particles.update(dt)

        for t in self.floating:
            t.update(dt)
        self.floating = [t for t in self.floating if t.life > 0]

        if self.state in ("PLAYING", "BOSS"):
            self.player.update(dt)

            self.camera.update(self.player.pos, dt)

            if self.state == "PLAYING":
                self.room_timer += dt
                self.spawn_timer -= dt

                if self.spawn_timer <= 0:
                    self.spawn_timer = max(.15, .85 - self.wave * .025)
                    self.spawn_enemy()

                for enemy in self.enemies:
                    enemy.update(dt)

                if self.room_timer > 32 and not self.enemies:
                    self.next_room()

            else:
                if self.boss:
                    self.boss.update(dt)

            for projectile in self.projectiles:
                projectile.update(dt)

                for enemy in self.enemies[:]:
                    if enemy.id in projectile.hit_ids:
                        continue

                    if projectile.pos.distance_to(enemy.pos) < projectile.radius + enemy.radius:
                        enemy.hit(projectile.damage, projectile)
                        projectile.hit_ids.add(enemy.id)

                        if len(projectile.hit_ids) > projectile.pierce:
                            projectile.life = 0
                        break

                if self.boss and projectile.life > 0:
                    if projectile.pos.distance_to(self.boss.pos) < projectile.radius + self.boss.radius:
                        self.boss.hit(projectile.damage)
                        projectile.life = 0

            for projectile in self.enemy_projectiles:
                projectile.update(dt)

                if projectile.pos.distance_to(self.player.pos) < projectile.radius + self.player.radius:
                    self.player.take_damage(projectile.damage)
                    projectile.life = 0

            self.projectiles = [
                p for p in self.projectiles
                if p.life > 0
                and -1400 < p.pos.x < 1400
                and -800 < p.pos.y < 800
            ]

            self.enemy_projectiles = [
                p for p in self.enemy_projectiles
                if p.life > 0
                and -1400 < p.pos.x < 1400
                and -800 < p.pos.y < 800
            ]

        elif self.state == "GAMEOVER":
            if self.score > self.high_score:
                self.high_score = self.score
                self.save()

    # ---------------------------- drawing ----------------------------

    def draw_background(self):
        screen.fill((5, 7, 15))

        # Grid
        offset = self.camera.pos

        grid = 80

        start_x = int(offset.x // grid) * grid
        start_y = int(offset.y // grid) * grid

        for x in range(start_x - 1600, start_x + 1600, grid):
            sx = x - offset.x + WIDTH / 2
            pygame.draw.line(screen, (12, 20, 34), (sx, 0), (sx, HEIGHT))

        for y in range(start_y - 1000, start_y + 1000, grid):
            sy = y - offset.y + HEIGHT / 2
            pygame.draw.line(screen, (12, 20, 34), (0, sy), (WIDTH, sy))

        # Arena
        center = pygame.Vector2(WIDTH / 2, HEIGHT / 2) - self.camera.pos
        rect = pygame.Rect(center.x - 1160, center.y - 610, 2320, 1220)

        pygame.draw.rect(screen, (15, 10, 30), rect, 4)

    def draw_hud(self):
        p = self.player

        # HP
        x, y = 30, 30
        w = 280

        pygame.draw.rect(screen, (18, 20, 30), (x, y, w, 18))
        pygame.draw.rect(
            screen,
            (255, 65, 100),
            (x, y, w * p.hp / p.max_hp, 18)
        )

        draw_text(
            screen,
            f"HP {int(p.hp)}/{int(p.max_hp)}",
            (x + 8, y + 1),
            SMALL,
            (255, 235, 240)
        )

        # XP
        y += 30

        pygame.draw.rect(screen, (18, 20, 30), (x, y, w, 8))
        pygame.draw.rect(
            screen,
            (70, 210, 255),
            (x, y, w * p.xp / p.xp_next, 8)
        )

        draw_text(
            screen,
            f"LV {p.level}",
            (x + w + 12, y - 6),
            SMALL,
            (80, 220, 255)
        )

        # Stats
        draw_text(
            screen,
            f"WAVE {self.wave:02d}   ROOM {self.room:02d}",
            (WIDTH - 250, 30),
            SMALL,
            (180, 190, 220)
        )

        draw_text(
            screen,
            f"SCORE {self.score:08d}",
            (WIDTH - 250, 55),
            SMALL,
            (255, 220, 130)
        )

        draw_text(
            screen,
            f"KILLS {self.kills:04d}",
            (WIDTH - 250, 80),
            SMALL,
            (190, 200, 220)
        )

        # Crosshair
        mouse = pygame.Vector2(pygame.mouse.get_pos())

        pygame.draw.circle(screen, (80, 230, 255), mouse, 9, 1)
        pygame.draw.line(
            screen,
            (80, 230, 255),
            mouse + pygame.Vector2(-14, 0),
            mouse + pygame.Vector2(-4, 0)
        )
        pygame.draw.line(
            screen,
            (80, 230, 255),
            mouse + pygame.Vector2(4, 0),
            mouse + pygame.Vector2(14, 0)
        )

    def draw(self):
        self.draw_background()

        offset = self.camera.offset()

        # World objects
        for projectile in self.projectiles:
            projectile.draw(screen, self.camera)

        for projectile in self.enemy_projectiles:
            projectile.draw(screen, self.camera)

        for enemy in self.enemies:
            enemy.draw(screen, self.camera)

        if self.boss:
            self.boss.draw(screen, self.camera)

        self.player.draw(screen, self.camera)

        self.particles.draw(screen, self.camera)

        for text in self.floating:
            text.draw(screen, self.camera)

        if self.state in ("PLAYING", "BOSS"):
            self.draw_hud()

        if self.state == "MENU":
            self.draw_menu()

        elif self.state == "UPGRADE":
            self.draw_upgrade()

        elif self.state == "GAMEOVER":
            self.draw_gameover()

        # Scanline effect
        overlay = pygame.Surface((WIDTH, HEIGHT), pygame.SRCALPHA)

        for y in range(0, HEIGHT, 4):
            pygame.draw.line(
                overlay,
                (255, 255, 255, 5),
                (0, y),
                (WIDTH, y)
            )

        screen.blit(overlay, (0, 0))

    def draw_menu(self):
        overlay = pygame.Surface((WIDTH, HEIGHT), pygame.SRCALPHA)
        overlay.fill((2, 4, 12, 190))
        screen.blit(overlay, (0, 0))

        draw_text(
            screen,
            "NEON//VOID",
            (WIDTH / 2, 210),
            HUGE,
            (90, 225, 255),
            True
        )

        draw_text(
            screen,
            "A PROCEDURAL CYBERPUNK ROGUELITE",
            (WIDTH / 2, 280),
            SMALL,
            (180, 190, 220),
            True
        )

        draw_text(
            screen,
            "[ ENTER ]  START",
            (WIDTH / 2, 385),
            BIG,
            (230, 240, 255),
            True
        )

        draw_text(
            screen,
            "WASD MOVE   •   LMB FIRE   •   SPACE DASH",
            (WIDTH / 2, 445),
            SMALL,
            (120, 140, 170),
            True
        )

        draw_text(
            screen,
            f"BEST SCORE  {self.high_score:08d}",
            (WIDTH / 2, 505),
            SMALL,
            (255, 210, 120),
            True
        )

    def draw_upgrade(self):
        overlay = pygame.Surface((WIDTH, HEIGHT), pygame.SRCALPHA)
        overlay.fill((3, 5, 15, 225))
        screen.blit(overlay, (0, 0))

        draw_text(
            screen,
            "SYSTEM UPGRADE",
            (WIDTH / 2, 105),
            HUGE,
            (90, 225, 255),
            True
        )

        draw_text(
            screen,
            "SELECT ONE AUGMENTATION",
            (WIDTH / 2, 175),
            SMALL,
            (150, 170, 200),
            True
        )

        card_w = 300
        card_h = 220
        gap = 30

        total = card_w * 3 + gap * 2
        start = (WIDTH - total) / 2

        mouse = pygame.Vector2(pygame.mouse.get_pos())

        for i, upgrade in enumerate(self.upgrade_choices):
            x = start + i * (card_w + gap)
            y = 245

            rect = pygame.Rect(x, y, card_w, card_h)
            hovered = rect.collidepoint(mouse)

            border = (90, 230, 255) if hovered else (40, 70, 100)
            bg = (12, 28, 45) if hovered else (10, 15, 28)

            pygame.draw.rect(screen, bg, rect)
            pygame.draw.rect(screen, border, rect, 2)

            draw_text(
                screen,
                f"[ {i + 1} ]",
                (x + 20, y + 18),
                SMALL,
                border
            )

            draw_text(
                screen,
                upgrade[0],
                (x + 20, y + 65),
                BIG,
                (225, 240, 255)
            )

            # wrap manually
            draw_text(
                screen,
                upgrade[1],
                (x + 20, y + 125),
                SMALL,
                (150, 180, 205)
            )

        draw_text(
            screen,
            "PRESS 1 / 2 / 3 OR CLICK A CARD",
            (WIDTH / 2, 535),
            SMALL,
            (120, 150, 180),
            True
        )

    def draw_gameover(self):
        overlay = pygame.Surface((WIDTH, HEIGHT), pygame.SRCALPHA)
        overlay.fill((8, 2, 10, 220))
        screen.blit(overlay, (0, 0))

        draw_text(
            screen,
            "SYSTEM FAILURE",
            (WIDTH / 2, 230),
            HUGE,
            (255, 65, 105),
            True
        )

        draw_text(
            screen,
            f"SCORE  {self.score:08d}",
            (WIDTH / 2, 330),
            BIG,
            (255, 225, 150),
            True
        )

        draw_text(
            screen,
            f"BEST   {self.high_score:08d}",
            (WIDTH / 2, 375),
            SMALL,
            (170, 180, 205),
            True
        )

        draw_text(
            screen,
            "[ ENTER ]  REBOOT",
            (WIDTH / 2, 470),
            BIG,
            (220, 230, 245),
            True
        )

    # ---------------------------- input ----------------------------

    def event(self, event):
        if event.type == pygame.KEYDOWN:

            if event.key == pygame.K_ESCAPE:
                if self.state == "PLAYING" or self.state == "BOSS":
                    self.state = "MENU"

            if self.state == "MENU":
                if event.key in (pygame.K_RETURN, pygame.K_SPACE):
                    self.start()

            elif self.state == "GAMEOVER":
                if event.key == pygame.K_RETURN:
                    self.start()

            elif self.state == "UPGRADE":
                if event.key in (pygame.K_1, pygame.K_KP1):
                    self.apply_upgrade(0)
                elif event.key in (pygame.K_2, pygame.K_KP2):
                    self.apply_upgrade(1)
                elif event.key in (pygame.K_3, pygame.K_KP3):
                    self.apply_upgrade(2)

            elif self.state in ("PLAYING", "BOSS"):
                if event.key == pygame.K_SPACE:
                    self.player.dash()

        if event.type == pygame.MOUSEBUTTONDOWN:
            if self.state == "UPGRADE" and event.button == 1:
                card_w = 300
                card_h = 220
                gap = 30
                total = card_w * 3 + gap * 2
                start = (WIDTH - total) / 2

                for i in range(3):
                    rect = pygame.Rect(
                        start + i * (card_w + gap),
                        245,
                        card_w,
                        card_h
                    )

                    if rect.collidepoint(event.pos):
                        self.apply_upgrade(i)


# ---------------------------------------------------------------------------
# Main
# ---------------------------------------------------------------------------

def main():
    game = Game()

    running = True

    while running:
        dt = min(clock.tick(FPS) / 1000.0, .033)

        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                running = False
            else:
                game.event(event)

        game.update(dt)
        game.draw()

        pygame.display.flip()

    game.save()
    pygame.quit()


if __name__ == "__main__":
    main()
