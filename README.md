"""
rocket_shooter.py
A simple rocket shooter game built with Pygame.
Controls:
  - Left / A : move left
  - Right / D : move right
  - Up / W : move up
  - Down / S : move down
  - Space : shoot
  - P : pause/unpause
  - R : restart after game over
"""

import pygame
import random
import os
import math
from dataclasses import dataclass

# --- Config ---
WIDTH, HEIGHT = 800, 600
FPS = 60
PLAYER_SPEED = 5
BULLET_SPEED = 10
ENEMY_MIN_SPEED = 1.5
ENEMY_MAX_SPEED = 3.0
ENEMY_SPAWN_RATE = 900  # milliseconds between spawn; lowered per level
LEVEL_UP_SCORE = 100  # points to level up initially
POWERUP_DURATION = 5000  # ms
HIGH_SCORE_FILE = "highscore.txt"

pygame.init()
screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("Rocket Shooter")
clock = pygame.time.Clock()
font = pygame.font.SysFont("consolas", 20)
big_font = pygame.font.SysFont("consolas", 48)


def load_highscore():
    if not os.path.exists(HIGH_SCORE_FILE):
        return 0
    try:
        with open(HIGH_SCORE_FILE, "r") as f:
            return int(f.read().strip() or 0)
    except Exception:
        return 0


def save_highscore(score):
    try:
        with open(HIGH_SCORE_FILE, "w") as f:
            f.write(str(score))
    except Exception:
        pass


# --- Entities ---
@dataclass
class Player:
    x: float
    y: float
    w: int = 36
    h: int = 48
    speed: float = PLAYER_SPEED
    lives: int = 3
    invulnerable_until: int = 0
    rapid_fire_until: int = 0

    def rect(self):
        return pygame.Rect(int(self.x - self.w // 2), int(self.y - self.h // 2), self.w, self.h)


@dataclass
class Bullet:
    x: float
    y: float
    vx: float = 0
    vy: float = -BULLET_SPEED
    owner: str = "player"  # could be 'enemy' too
    w: int = 4
    h: int = 10

    def rect(self):
        return pygame.Rect(int(self.x - self.w // 2), int(self.y - self.h // 2), self.w, self.h)


@dataclass
class Enemy:
    x: float
    y: float
    vx: float
    vy: float
    w: int = 36
    h: int = 36
    hp: int = 1
    score: int = 10

    def rect(self):
        return pygame.Rect(int(self.x - self.w // 2), int(self.y - self.h // 2), self.w, self.h)


@dataclass
class PowerUp:
    x: float
    y: float
    kind: str  # 'rapid'
    vy: float = 2
    w: int = 20
    h: int = 20

    def rect(self):
        return pygame.Rect(int(self.x - self.w // 2), int(self.y - self.h // 2), self.w, self.h)


# --- Game state ---
def new_game_state():
    return {
        "player": Player(WIDTH // 2, HEIGHT - 80),
        "bullets": [],
        "enemies": [],
        "enemy_bullets": [],
        "powerups": [],
        "score": 0,
        "level": 1,
        "enemy_spawn_timer": 0,
        "spawn_interval": ENEMY_SPAWN_RATE,
        "last_shot": 0,
        "shot_cooldown": 300,  # ms
        "running": True,
        "paused": False,
        "game_over": False,
        "highscore": load_highscore(),
        "next_level_score": LEVEL_UP_SCORE,
    }


state = new_game_state()


# --- Helper drawing functions ---
def draw_player(surf, player: Player):
    r = player.rect()
    # Rocket body (triangle)
    points = [
        (player.x, player.y - player.h // 2),
        (player.x - player.w // 2, player.y + player.h // 2),
        (player.x + player.w // 2, player.y + player.h // 2),
    ]
    color = (200, 200, 255) if pygame.time.get_ticks() < player.invulnerable_until else (100, 180, 255)
    pygame.draw.polygon(surf, color, points)
    # windows / detail
    pygame.draw.circle(surf, (220, 220, 255), (int(player.x), int(player.y - 6)), 6)


def draw_bullet(surf, b: Bullet):
    pygame.draw.rect(surf, (255, 240, 100), b.rect())


def draw_enemy(surf, e: Enemy):
    rect = e.rect()
    # simple "alien" as circle with eyes
    pygame.draw.ellipse(surf, (220, 100, 100), rect)
    eye_y = int(e.y - e.h * 0.12)
    pygame.draw.circle(surf, (255, 255, 255), (int(e.x - e.w * 0.17), eye_y), 4)
    pygame.draw.circle(surf, (255, 255, 255), (int(e.x + e.w * 0.17), eye_y), 4)


def draw_powerup(surf, p: PowerUp):
    rect = p.rect()
    pygame.draw.rect(surf, (100, 255, 100), rect)
    txt = font.render("R", True, (0, 80, 0))
    surf.blit(txt, (p.x - txt.get_width() // 2, p.y - txt.get_height() // 2))


# --- Spawn logic ---
def spawn_enemy(level):
    # spawn from top at random x. speed scales with level.
    x = random.randint(40, WIDTH - 40)
    y = -30
    speed = random.uniform(ENEMY_MIN_SPEED + level * 0.15, ENEMY_MAX_SPEED + level * 0.2)
    # give an initial horizontal drift
    vx = random.uniform(-1.2 - level * 0.05, 1.2 + level * 0.05)
    hp = 1 + (level // 3)  # tougher enemies every few levels
    score = 10 + (level - 1) * 2
    return Enemy(x, y, vx, speed, hp=hp, score=score)


def spawn_powerup(x, y):
    # Only 'rapid' implemented currently
    return PowerUp(x, y, kind="rapid")


# --- Collision helpers ---
def collide_rect(a, b):
    return a.rect().colliderect(b.rect())


# --- Game update loop step ---
def update(dt):
    if state["paused"] or state["game_over"]:
        return

    player: Player = state["player"]
    now = pygame.time.get_ticks()

    # spawn enemies
    state["enemy_spawn_timer"] += dt
    if state["enemy_spawn_timer"] >= state["spawn_interval"]:
        state["enemy_spawn_timer"] = 0
        state["enemies"].append(spawn_enemy(state["level"]))

    # Move bullets
    for b in list(state["bullets"]):
        b.x += b.vx
        b.y += b.vy
        if b.y < -20 or b.y > HEIGHT + 20 or b.x < -20 or b.x > WIDTH + 20:
            state["bullets"].remove(b)

    # Move enemies
    for e in list(state["enemies"]):
        e.x += e.vx
        e.y += e.vy
        # bounce horizontally
        if e.x < 20:
            e.x = 20
            e.vx = -e.vx
        if e.x > WIDTH - 20:
            e.x = WIDTH - 20
            e.vx = -e.vx

        # enemy off-screen bottom -> player loses life
        if e.y > HEIGHT + 40:
            state["enemies"].remove(e)
            player.lives -= 1
            player.invulnerable_until = now + 1500
            if player.lives <= 0:
                trigger_game_over()
        # simple enemy shooting chance
        if random.random() < 0.002 + 0.001 * state["level"]:
            # fire enemy bullet downwards
            bx = e.x
            by = e.y + e.h // 2 + 6
            state["enemy_bullets"].append(Bullet(bx, by, 0, 4, owner="enemy"))

    # Move enemy bullets
    for b in list(state["enemy_bullets"]):
        b.x += b.vx
        b.y += b.vy
        if b.y < -20 or b.y > HEIGHT + 20:
            state["enemy_bullets"].remove(b)

    # Powerups movement
    for p in list(state["powerups"]):
        p.y += p.vy
        if p.y > HEIGHT + 30:
            state["powerups"].remove(p)

    # Check collisions: bullets -> enemies
    for b in list(state["bullets"]):
        for e in list(state["enemies"]):
            if b.owner == "player" and collide_rect(b, e):
                state["bullets"].remove(b)
                e.hp -= 1
                if e.hp <= 0:
                    # enemy dies
                    state["score"] += e.score
                    # drop powerup sometimes
                    if random.random() < 0.12:
                        state["powerups"].append(spawn_powerup(e.x, e.y))
                    try:
                        state["enemies"].remove(e)
                    except ValueError:
                        pass
                break

    # Enemy bullets -> player
    for b in list(state["enemy_bullets"]):
        if collide_rect(b, player) and pygame.time.get_ticks() >= player.invulnerable_until:
            try:
                state["enemy_bullets"].remove(b)
            except ValueError:
                pass
            player.lives -= 1
            player.invulnerable_until = now + 1500
            if player.lives <= 0:
                trigger_game_over()

    # Enemies collide with player
    for e in list(state["enemies"]):
        if collide_rect(e, player) and pygame.time.get_ticks() >= player.invulnerable_until:
            try:
                state["enemies"].remove(e)
            except ValueError:
                pass
            player.lives -= 1
            player.invulnerable_until = now + 1500
            if player.lives <= 0:
                trigger_game_over()

    # Player collects powerups
    for p in list(state["powerups"]):
        if collide_rect(p, player):
            if p.kind == "rapid":
                player.rapid_fire_until = now + POWERUP_DURATION
                state["shot_cooldown"] = 100  # faster shooting
            try:
                state["powerups"].remove(p)
            except ValueError:
                pass

    # Expire powerup
    if pygame.time.get_ticks() > player.rapid_fire_until:
        state["shot_cooldown"] = 300

    # Level up logic
    if state["score"] >= state["next_level_score"]:
        state["level"] += 1
        # increase spawn rate and enemy difficulty
        state["spawn_interval"] = max(250, int(state["spawn_interval"] * 0.88))
        state["next_level_score"] += LEVEL_UP_SCORE + state["level"] * 30

    # Keep bullets list bounded
    if len(state["bullets"]) > 100:
        state["bullets"] = state["bullets"][-100:]


def trigger_game_over():
    state["game_over"] = True
    if state["score"] > state["highscore"]:
        state["highscore"] = state["score"]
        save_highscore(state["highscore"])


# --- Input handling ---
def handle_input():
    keys = pygame.key.get_pressed()
    player: Player = state["player"]

    # Movement
    dx = 0
    dy = 0
    if keys[pygame.K_LEFT] or keys[pygame.K_a]:
        dx = -1
    if keys[pygame.K_RIGHT] or keys[pygame.K_d]:
        dx = 1
    if keys[pygame.K_UP] or keys[pygame.K_w]:
        dy = -1
    if keys[pygame.K_DOWN] or keys[pygame.K_s]:
        dy = 1

    # normalize diagonal speed
    if dx != 0 and dy != 0:
        dx *= 0.7071
        dy *= 0.7071

    player.x += dx * player.speed
    player.y += dy * player.speed

    # clamp to screen edges
    player.x = max(20, min(WIDTH - 20, player.x))
    player.y = max(20, min(HEIGHT - 20, player.y))

    # Shooting
    now = pygame.time.get_ticks()
    if (keys[pygame.K_SPACE] or keys[pygame.K_k]) and now - state["last_shot"] >= state["shot_cooldown"]:
        shoot_player()
        state["last_shot"] = now


def shoot_player():
    p = state["player"]
    # fire a spread if rapid fire active
    if pygame.time.get_ticks() < p.rapid_fire_until:
        # triple shot
        state["bullets"].append(Bullet(p.x, p.y - p.h // 2, -1.5, -BULLET_SPEED))
        state["bullets"].append(Bullet(p.x, p.y - p.h // 2, 0, -BULLET_SPEED))
        state["bullets"].append(Bullet(p.x, p.y - p.h // 2, 1.5, -BULLET_SPEED))
    else:
        state["bullets"].append(Bullet(p.x, p.y - p.h // 2, 0, -BULLET_SPEED))


# --- Drawing HUD and screens ---
def draw_ui(surf):
    # score, lives, level
    score_s = font.render(f"Score: {state['score']}", True, (255, 255, 255))
    surf.blit(score_s, (10, 10))
    hs = font.render(f"High: {state['highscore']}", True, (255, 255, 255))
    surf.blit(hs, (10, 36))
    level_s = font.render(f"Level: {state['level']}", True, (255, 255, 255))
    surf.blit(level_s, (WIDTH - 110, 10))
    lives_s = font.render(f"Lives: {state['player'].lives}", True, (255, 255, 255))
    surf.blit(lives_s, (WIDTH - 110, 36))
    if state["paused"]:
        pause_s = big_font.render("PAUSED", True, (255, 255, 0))
        surf.blit(pause_s, (WIDTH // 2 - pause_s.get_width() // 2, HEIGHT // 2 - 30))
    if state["game_over"]:
        go_s = big_font.render("GAME OVER", True, (255, 80, 80))
        surf.blit(go_s, (WIDTH // 2 - go_s.get_width() // 2, HEIGHT // 2 - 40))
        info = font.render("Press R to restart", True, (255, 255, 255))
        surf.blit(info, (WIDTH // 2 - info.get_width() // 2, HEIGHT // 2 + 18))


# --- Main loop ---
def main():
    running = True
    last_time = pygame.time.get_ticks()

    while running:
        dt = clock.tick(FPS)
        now = pygame.time.get_ticks()
        # events
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                running = False

            if event.type == pygame.KEYDOWN:
                if event.key == pygame.K_p:
                    state["paused"] = not state["paused"]
                if event.key == pygame.K_r:
                    # restart
                    if state["game_over"]:
                        restart_game()
                # quick debug spawn powerup
                if event.key == pygame.K_u:
                    state["powerups"].append(spawn_powerup(random.randint(40, WIDTH - 40), -10))

        handle_input()
        update(dt)

        # Drawing
        screen.fill((8, 12, 30))  # dark background

        # stars background - subtle parallax grid
        for i in range(30):
            sx = (i * 37 + (now // 8) % 200) % WIDTH
            sy = (i * 23 + (now // 10) % 200) % HEIGHT
            pygame.draw.circle(screen, (20, 20, 40), (sx, sy), 1)

        # draw entities
        for b in state["bullets"]:
            draw_bullet(screen, b)
        for b in state["enemy_bullets"]:
            pygame.draw.rect(screen, (230, 80, 80), b.rect())
        for e in state["enemies"]:
            draw_enemy(screen, e)
        for p in state["powerups"]:
            draw_powerup(screen, p)

        draw_player(screen, state["player"])

        draw_ui(screen)

        pygame.display.flip()

    pygame.quit()


def restart_game():
    global state
    state = new_game_state()


if __name__ == "__main__":
    main()
