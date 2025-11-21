# Panel
"""
Flappy Bird - Single-file professional Python implementation using Pygame.
No external images or sounds required (everything drawn with primitives).
Controls:
    - Space / Up arrow / Left mouse click: flap
    - R: restart after game over
Requirements:
    - pygame (install with: pip install pygame)
Author: Generated for user
"""

import sys
import math
import random
import dataclasses
from typing import Tuple, List

import pygame

# ---------------------------
# Configuration / Constants
# ---------------------------
SCREEN_WIDTH = 400
SCREEN_HEIGHT = 600
FPS = 60

GRAVITY = 900.0            # pixels / s^2
FLAP_VELOCITY = -300.0     # pixels / s
MAX_DROP_SPEED = 800.0

BIRD_RADIUS = 12
BIRD_X = SCREEN_WIDTH * 0.2

PIPE_WIDTH = 70
PIPE_GAP = 150             # vertical gap between top and bottom pipes
PIPE_SPEED_START = 140     # pixels / s
PIPE_DISTANCE = 220        # horizontal distance between consecutive pipes
PIPE_VARIATION = 120       # how much vertical center can vary
PIPE_MIN_Y = 80
PIPE_MAX_Y = SCREEN_HEIGHT - 120

GROUND_HEIGHT = 100
SCROLLING_GROUND_SPEED = 160

FONT_NAME = None  # default pygame font
SCORE_FILE = "flappy_highscore.txt"


# ---------------------------
# Utility Data Classes
# ---------------------------
@dataclasses.dataclass
class BirdState:
    y: float
    vy: float = 0.0
    rotation: float = 0.0  # degrees


@dataclasses.dataclass
class Pipe:
    x: float
    center_y: float

    @property
    def top(self) -> float:
        return self.center_y - PIPE_GAP / 2

    @property
    def bottom(self) -> float:
        return self.center_y + PIPE_GAP / 2


# ---------------------------
# Game Class
# ---------------------------
class FlappyGame:
    def __init__(self, screen: pygame.Surface):
        self.screen = screen
        self.clock = pygame.time.Clock()
        self.font = pygame.font.Font(FONT_NAME, 28)
        self.small_font = pygame.font.Font(FONT_NAME, 18)

        self.reset_game()
        self.load_highscore()

    def reset_game(self):
        self.bird = BirdState(y=SCREEN_HEIGHT / 2)
        self.pipes: List[Pipe] = []
        self.spawn_timer = 0.0
        self.score = 0
        self.best_score = 0
        self.game_over = False
        self.started = False
        self.scroll_x = 0.0
        self.pipe_speed = PIPE_SPEED_START
        self._init_pipes()

    def _init_pipes(self):
        # spawn a few pipes offscreen to the right
        self.pipes.clear()
        start_x = SCREEN_WIDTH + 60
        for i in range(3):
            cx = start_x + i * PIPE_DISTANCE
            center_y = random.uniform(PIPE_MIN_Y + PIPE_GAP / 2, PIPE_MAX_Y - PIPE_GAP / 2)
            self.pipes.append(Pipe(x=cx, center_y=center_y))

    # ---------------------------
    # Persistence
    # ---------------------------
    def load_highscore(self):
        try:
            with open(SCORE_FILE, "r", encoding="utf-8") as f:
                self.best_score = int(f.read().strip() or 0)
        except Exception:
            self.best_score = 0

    def save_highscore(self):
        try:
            if self.score > self.best_score:
                with open(SCORE_FILE, "w", encoding="utf-8") as f:
                    f.write(str(self.score))
                self.best_score = self.score
        except Exception:
            pass

    # ---------------------------
    # Input
    # ---------------------------
    def handle_event(self, event: pygame.event.Event):
        if event.type == pygame.QUIT:
            self.terminate()
        if event.type == pygame.KEYDOWN:
            if event.key in (pygame.K_SPACE, pygame.K_UP):
                self.flap()
            if event.key == pygame.K_r and self.game_over:
                self.reset_game()
        if event.type == pygame.MOUSEBUTTONDOWN:
            if event.button == 1:
                self.flap()
        # allow touch-like events
        if event.type == pygame.FINGERDOWN:
            self.flap()

    def flap(self):
        if self.game_over:
            return
        self.started = True
        self.bird.vy = FLAP_VELOCITY

    # ---------------------------
    # Physics and Game Update
    # ---------------------------
    def update(self, dt: float):
        if self.game_over:
            # gentle falling animation after death
            self.bird.vy = min(self.bird.vy + GRAVITY * dt, MAX_DROP_SPEED)
            self.bird.y += self.bird.vy * dt
            self.bird.rotation = min(90.0, self.bird.rotation + 200.0 * dt)
            return

        if not self.started:
            # subtle bob before start
            self.bird.y = SCREEN_HEIGHT / 2 + math.sin(pygame.time.get_ticks() / 400.0) * 6
            return

        # Apply physics
        self.bird.vy = min(self.bird.vy + GRAVITY * dt, MAX_DROP_SPEED)
        self.bird.y += self.bird.vy * dt

        # rotation: tilt up when rising, down when falling
        target_rot = -25.0 if self.bird.vy < 0 else min(90.0, (self.bird.vy / MAX_DROP_SPEED) * 90.0)
        # smooth rotation
        self.bird.rotation += (target_rot - self.bird.rotation) * min(1.0, 12.0 * dt)

        # Move pipes and ground
        dx = self.pipe_speed * dt
        for pipe in self.pipes:
            pipe.x -= dx
        self.scroll_x += SCROLLING_GROUND_SPEED * dt

        # Remove off-screen pipes and spawn new ones
        if self.pipes and self.pipes[0].x < -PIPE_WIDTH - 20:
            self.pipes.pop(0)
        # Ensure we always have at least 3 pipes ahead
        while len(self.pipes) < 4:
            last_x = self.pipes[-1].x if self.pipes else SCREEN_WIDTH
            new_x = last_x + PIPE_DISTANCE
            # vary center smoothly from previous to keep it playable
            prev_center = self.pipes[-1].center_y if self.pipes else SCREEN_HEIGHT / 2
            variation = random.uniform(-PIPE_VARIATION / 2, PIPE_VARIATION / 2)
            center_y = max(PIPE_MIN_Y + PIPE_GAP / 2, min(PIPE_MAX_Y - PIPE_GAP / 2, prev_center + variation))
            self.pipes.append(Pipe(x=new_x, center_y=center_y))

        # Scoring: if bird passes a pipe's x coordinate, increase score once
        for pipe in self.pipes:
            if not getattr(pipe, "_scored", False) and pipe.x + PIPE_WIDTH / 2 < BIRD_X:
                setattr(pipe, "_scored", True)
                self.score += 1
                # ramp up speed slightly every few points
                if self.score % 5 == 0:
                    self.pipe_speed += 6

        # Collisions
        self.check_collisions()

    def check_collisions(self):
        # ground
        ground_y = SCREEN_HEIGHT - GROUND_HEIGHT
        if self.bird.y + BIRD_RADIUS >= ground_y:
            self.bird.y = ground_y - BIRD_RADIUS
            self.trigger_game_over()
            return

        # ceiling
        if self.bird.y - BIRD_RADIUS <= 0:
            self.bird.y = BIRD_RADIUS
            self.bird.vy = 0.0

        # pipes: approximate circle-rectangle collisions
        bx, by, br = BIRD_X, self.bird.y, BIRD_RADIUS
        for pipe in self.pipes:
            px = pipe.x
            # top rect
            top_rect = pygame.Rect(int(px), 0, PIPE_WIDTH, int(pipe.top))
            bottom_rect = pygame.Rect(int(px), int(pipe.bottom), PIPE_WIDTH, SCREEN_HEIGHT - int(pipe.bottom) - GROUND_HEIGHT)
            if circle_rect_collision((bx, by), br, top_rect) or circle_rect_collision((bx, by), br, bottom_rect):
                self.trigger_game_over()
                return

    def trigger_game_over(self):
        if not self.game_over:
            self.game_over = True
            self.save_highscore()

    # ---------------------------
    # Drawing
    # ---------------------------
    def draw(self):
        self.screen.fill((135, 206, 235))  # sky blue

        # Draw pipes
        for pipe in self.pipes:
            self.draw_pipe(pipe)

        # Draw ground (two segments for scrolling illusion)
        ground_y = SCREEN_HEIGHT - GROUND_HEIGHT
        ground_rect = pygame.Rect(0, ground_y, SCREEN_WIDTH, GROUND_HEIGHT)
        pygame.draw.rect(self.screen, (222, 184, 135), ground_rect)  # sandy ground
        # ground stripes
        tile_w = 40
        offset = int(self.scroll_x % tile_w)
        for i in range(-1, SCREEN_WIDTH // tile_w + 2):
            x = i * tile_w - offset
            pygame.draw.rect(self.screen, (205, 133, 63), (x, ground_y + GROUND_HEIGHT - 20, tile_w // 2, 20))

        # Draw bird (circle with beak)
        self.draw_bird()

        # HUD
        self.draw_hud()

        # Game over overlay
        if self.game_over:
            self.draw_game_over()

    def draw_bird(self):
        # Bird body
        bx = int(BIRD_X)
        by = int(self.bird.y)
        # apply rotation to a simple triangular wing/beak shape
        body_color = (255, 215, 0)
        eye_color = (0, 0, 0)
        pygame.draw.circle(self.screen, body_color, (bx, by), BIRD_RADIUS)
        # beak - rotated triangle
        beak_length = 12
        angle = math.radians(self.bird.rotation)
        # symmetrical triangle in local coords, then rotate & translate
        points_local = [(BIRD_RADIUS, 0), (BIRD_RADIUS + beak_length, -6), (BIRD_RADIUS + beak_length, 6)]
        points = []
        for px, py in points_local:
            rx = px * math.cos(angle) - py * math.sin(angle)
            ry = px * math.sin(angle) + py * math.cos(angle)
            points.append((int(bx + rx), int(by + ry)))
        pygame.draw.polygon(self.screen, (255, 140, 0), points)
        # eye
        eye_offset = (-4, -6)
        ex = int(bx + eye_offset[0])
        ey = int(by + eye_offset[1])
        pygame.draw.circle(self.screen, eye_color, (ex, ey), 2)

    def draw_pipe(self, pipe: Pipe):
        px = int(pipe.x)
        # top pipe
        top_rect = pygame.Rect(px, 0, PIPE_WIDTH, int(pipe.top))
        bottom_rect = pygame.Rect(px, int(pipe.bottom), PIPE_WIDTH, SCREEN_HEIGHT - int(pipe.bottom))
        # pipe body
        pygame.draw.rect(self.screen, (34, 139, 34), top_rect)
        pygame.draw.rect(self.screen, (34, 139, 34), bottom_rect)
        # pipe cap (rounded look) - draw small darker rectangle at ends
        cap_height = 14
        pygame.draw.rect(self.screen, (0, 100, 0), (px - 2, int(pipe.top) - cap_height, PIPE_WIDTH + 4, cap_height))
        pygame.draw.rect(self.screen, (0, 100, 0), (px - 2, int(pipe.bottom), PIPE_WIDTH + 4, cap_height))

    def draw_hud(self):
        # Score top-center
        score_surf = self.font.render(str(self.score), True, (255, 255, 255))
        score_bg = pygame.Surface((score_surf.get_width() + 20, score_surf.get_height() + 12), pygame.SRCALPHA)
        score_bg.fill((0, 0, 0, 120))
        x = (SCREEN_WIDTH - score_bg.get_width()) // 2
        self.screen.blit(score_bg, (x, 20))
        self.screen.blit(score_surf, (x + 10, 26))

        # Best score bottom-left
        best_text = f"Best: {max(self.best_score, self.score)}"
        best_surf = self.small_font.render(best_text, True, (0, 0, 0))
        self.screen.blit(best_surf, (10, SCREEN_HEIGHT - GROUND_HEIGHT + 8))

        # Instructions (before start)
        if not self.started and not self.game_over:
            title = "Flappy"
            subtitle = "Press SPACE or Click to start"
            t_surf = self.font.render(title, True, (255, 255, 255))
            s_surf = self.small_font.render(subtitle, True, (255, 255, 255))
            # shadow for clarity
            self.screen.blit(t_surf, ((SCREEN_WIDTH - t_surf.get_width()) // 2, SCREEN_HEIGHT // 2 - 80))
            self.screen.blit(s_surf, ((SCREEN_WIDTH - s_surf.get_width()) // 2, SCREEN_HEIGHT // 2 - 40))

    def draw_game_over(self):
        overlay = pygame.Surface((SCREEN_WIDTH, SCREEN_HEIGHT), pygame.SRCALPHA)
        overlay.fill((0, 0, 0, 120))
        self.screen.blit(overlay, (0, 0))
        go_text = "Game Over"
        info_text = "Press R to Restart"
        go_surf = self.font.render(go_text, True, (255, 255, 255))
        info_surf = self.small_font.render(info_text, True, (255, 255, 255))
        score_surf = self.small_font.render(f"Score: {self.score}  Best: {self.best_score}", True, (255, 255, 255))
        self.screen.blit(go_surf, ((SCREEN_WIDTH - go_surf.get_width()) // 2, SCREEN_HEIGHT // 2 - 40))
        self.screen.blit(score_surf, ((SCREEN_WIDTH - score_surf.get_width()) // 2, SCREEN_HEIGHT // 2))
        self.screen.blit(info_surf, ((SCREEN_WIDTH - info_surf.get_width()) // 2, SCREEN_HEIGHT // 2 + 30))

    # ---------------------------
    # Game Loop
    # ---------------------------
    def run(self):
        while True:
            dt = self.clock.tick(FPS) / 1000.0
            for event in pygame.event.get():
                self.handle_event(event)

            self.update(dt)
            self.draw()
            pygame.display.flip()

    def terminate(self):
        pygame.quit()
        sys.exit()


# ---------------------------
# Helpers
# ---------------------------
def circle_rect_collision(circle_pos: Tuple[float, float], r: float, rect: pygame.Rect) -> bool:
    """Check collision between a circle and axis-aligned rectangle."""
    cx, cy = circle_pos
    nearest_x = max(rect.left, min(cx, rect.right))
    nearest_y = max(rect.top, min(cy, rect.bottom))
    dx = cx - nearest_x
    dy = cy - nearest_y
    return (dx * dx + dy * dy) <= (r * r)


# ---------------------------
# Entry Point
# ---------------------------
def main():
    pygame.init()
    pygame.display.set_caption("Flappy - Pygame")
    screen = pygame.display.set_mode((SCREEN_WIDTH, SCREEN_HEIGHT))
    game = FlappyGame(screen)
    try:
        game.run()
    except SystemExit:
        raise
    except Exception:
        pygame.quit()
        raise


if __name__ == "__main__":
    main()
