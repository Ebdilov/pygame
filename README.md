import pygame
import sys
from pathlib import Path

# --- Konfiqurasiya ---
WIDTH, HEIGHT = 960, 640
FPS = 60
TILE = 48
GRAVITY = 0.9
JUMP_POWER = -16
PLAYER_SPEED = 6

# Rənglər (dəyişə bilər)
WHITE = (255, 255, 255)
SKY = (92, 148, 252)
DARK = (20, 20, 20)

# Səviyyə xəritəsi (sadə)
# . = boşluq, # = platform, P = baslangic, E = dusmen, C = coin, G = finis
LEVEL = [
    '........................................................',
    '........................................................',
    '........................................................',
    '........................................................',
    '............................C...........................',
    '...............###......................................',
    '..................................................C.....',
    '......P..............##......................###........',
    '############################.....###########............',
    '................................#.......................',
    '..................E.....................................',
    '..............#####.............................G.......',
    '###############################..########################',
]

# --- Fayl yoxlanışı (assets varsa istifadə et) ---
ASSET_DIR = Path('assets')
PLAYER_IMG = ASSET_DIR / 'player.png'
ENEMY_IMG = ASSET_DIR / 'enemy.png'
COIN_IMG = ASSET_DIR / 'coin.png'

# --- Sprite sinifləri ---
class Camera:
    def __init__(self, width, height):
        self.offset = pygame.Vector2(0, 0)
        self.width = width
        self.height = height

    def follow(self, target_rect):
        # Mərkəzə yaxın saxla
        self.offset.x = target_rect.centerx - WIDTH // 2
        self.offset.y = target_rect.centery - HEIGHT // 2
        # Sıfırdan kiçik olmasın (sol və üst üçün)
        if self.offset.x < 0:
            self.offset.x = 0
        if self.offset.y < 0:
            self.offset.y = 0

class Platform(pygame.sprite.Sprite):
    def __init__(self, x, y, w=TILE, h=TILE):
        super().__init__()
        self.image = pygame.Surface((w, h))
        self.image.fill((100, 60, 30))
        self.rect = self.image.get_rect(topleft=(x, y))

class Coin(pygame.sprite.Sprite):
    def __init__(self, x, y):
        super().__init__()
        size = TILE // 2
        if COIN_IMG.exists():
            self.image = pygame.image.load(COIN_IMG).convert_alpha()
            self.image = pygame.transform.scale(self.image, (size, size))
        else:
            self.image = pygame.Surface((size, size), pygame.SRCALPHA)
            pygame.draw.circle(self.image, (255, 215, 0), (size//2, size//2), size//2)
        self.rect = self.image.get_rect(center=(x + TILE//2, y + TILE//2))

class Enemy(pygame.sprite.Sprite):
    def __init__(self, x, y):
        super().__init__()
        if ENEMY_IMG.exists():
            self.image = pygame.image.load(ENEMY_IMG).convert_alpha()
            self.image = pygame.transform.scale(self.image, (TILE - 8, TILE - 12))
        else:
            self.image = pygame.Surface((TILE - 8, TILE - 12))
            self.image.fill((180, 50, 50))
        self.rect = self.image.get_rect(topleft=(x, y))
        self.vx = 2

    def update(self, platforms):
        self.rect.x += self.vx
        # Düşməni platformlardan atlamaması üçün yoxla ayağı altında platform varmı
        hits = pygame.sprite.spritecollide(self, platforms, False)
        if hits:
            # sadə davranış: tərs istiqamətə dön
            collide = False
            for p in hits:
                if self.vx > 0 and self.rect.right > p.rect.right:
                    collide = True
                if self.vx < 0 and self.rect.left < p.rect.left:
                    collide = True
            if collide:
                self.vx *= -1
        # Sərhəd yoxu: əgər önündə boşluq varsa da dön
        self.rect.y += 1
        under = pygame.sprite.spritecollide(self, platforms, False)
        self.rect.y -= 1
        if not under:
            self.vx *= -1

class Player(pygame.sprite.Sprite):
    def __init__(self, x, y):
        super().__init__()
        if PLAYER_IMG.exists():
            self.image = pygame.image.load(PLAYER_IMG).convert_alpha()
            self.image = pygame.transform.scale(self.image, (TILE - 12, TILE - 6))
        else:
            self.image = pygame.Surface((TILE - 12, TILE - 6))
            self.image.fill((50, 120, 50))
        self.rect = self.image.get_rect(topleft=(x, y))
        self.vx = 0
        self.vy = 0
        self.on_ground = False
        self.facing_right = True
        self.coins = 0

    def handle_input(self, keys):
        self.vx = 0
        if keys[pygame.K_LEFT] or keys[pygame.K_a]:
            self.vx = -PLAYER_SPEED
            self.facing_right = False
        if keys[pygame.K_RIGHT] or keys[pygame.K_d]:
            self.vx = PLAYER_SPEED
            self.facing_right = True

    def jump(self):
        if self.on_ground:
            self.vy = JUMP_POWER
            self.on_ground = False

    def apply_gravity(self):
        self.vy += GRAVITY
        if self.vy > 25:
            self.vy = 25

    def update(self, platforms):
        # horizontal
        self.rect.x += self.vx
        hits = pygame.sprite.spritecollide(self, platforms, False)
        for p in hits:
            if self.vx > 0:
                self.rect.right = p.rect.left
            elif self.vx < 0:
                self.rect.left = p.rect.right
        # vertical
        self.apply_gravity()
        self.rect.y += self.vy
        hits = pygame.sprite.spritecollide(self, platforms, False)
        self.on_ground = False
        for p in hits:
            if self.vy > 0:
                self.rect.bottom = p.rect.top
                self.vy = 0
                self.on_ground = True
            elif self.vy < 0:
                self.rect.top = p.rect.bottom
                self.vy = 0

# --- Level loader ---
class Level:
    def __init__(self, level_map):
        self.platforms = pygame.sprite.Group()
        self.coins = pygame.sprite.Group()
        self.enemies = pygame.sprite.Group()
        self.all_sprites = pygame.sprite.Group()
        self.player = None
        self.goal = None
        self.width = len(level_map[0]) * TILE
        self.height = len(level_map) * TILE
        self.parse(level_map)

    def parse(self, level_map):
        for row, line in enumerate(level_map):
            for col, ch in enumerate(line):
                x = col * TILE
                y = row * TILE
                if ch == '#':
                    p = Platform(x, y, TILE, TILE)
                    self.platforms.add(p)
                    self.all_sprites.add(p)
                elif ch == 'P':
                    self.player = Player(x, y - 2)
                    self.all_sprites.add(self.player)
                elif ch == 'C':
                    c = Coin(x, y)
                    self.coins.add(c)
                    self.all_sprites.add(c)
                elif ch == 'E':
                    e = Enemy(x, y)
                    self.enemies.add(e)
                    self.all_sprites.add(e)
                elif ch == 'G':
                    # Goal — böyük qutu
                    g = pygame.Surface((TILE, TILE * 2))
                    g.fill((120, 200, 120))
                    sprite = pygame.sprite.Sprite()
                    sprite.image = g
                    sprite.rect = g.get_rect(topleft=(x, y - TILE))
                    self.goal = sprite
                    self.all_sprites.add(sprite)

# --- Oyunun əsas hissəsi ---
def main():
    pygame.init()
    screen = pygame.display.set_mode((WIDTH, HEIGHT))
    pygame.display.set_caption('Mini Mario - Python Pygame')
    clock = pygame.time.Clock()

    level = Level(LEVEL)
    if not level.player:
        print('Səviyyədə P (player) tapılmadı!')
        pygame.quit()
        sys.exit()

    camera = Camera(level.width, level.height)

    font = pygame.font.SysFont(None, 28)

    running = True
    while running:
        dt = clock.tick(FPS) / 1000
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                running = False
            elif event.type == pygame.KEYDOWN:
                if event.key == pygame.K_SPACE or event.key == pygame.K_w or event.key == pygame.K_UP:
                    level.player.jump()

        keys = pygame.key.get_pressed()
        level.player.handle_input(keys)

        # update
        level.player.update(level.platforms)
        for e in list(level.enemies):
            e.update(level.platforms)
            # enemy-player collision -> sadə ölüm: player arxaya sıçrar
            if e.rect.colliderect(level.player.rect):
                # əgər player yuxarıdan düşübdirsə, düşməni öldür
                if level.player.vy > 0 and level.player.rect.bottom <= e.rect.top + 12:
                    level.enemies.remove(e)
                    level.all_sprites.remove(e)
                    level.player.vy = JUMP_POWER / 2
                else:
                    # reset player to start (sadə)
                    # alternativ: həyat/ölüm menyusu
                    level = Level(LEVEL)
                    camera = Camera(level.width, level.height)
                    continue

        # coin collection
        collected = pygame.sprite.spritecollide(level.player, level.coins, dokill=True)
        for c in collected:
            level.player.coins += 1

        # check goal
        if level.goal and level.player.rect.colliderect(level.goal.rect):
            print('Level tamamlandı! Pul sayi:', level.player.coins)
            # restart or quit
            running = False

        # kamera follow
        camera.follow(level.player.rect)

        # draw
        screen.fill(SKY)
        # parallax ground (sadə)
        pygame.draw.rect(screen, (70, 200, 70), (-camera.offset.x, HEIGHT - 60 - camera.offset.y, level.width, 200))

        for sprite in level.all_sprites:
            # sprite.position - camera.offset
            rect = sprite.rect.move(-camera.offset.x, -camera.offset.y)
            screen.blit(sprite.image, rect)

        # HUD
        hud = font.render(f'Pul: {level.player.coins}', True, DARK)
        screen.blit(hud, (10, 10))

        pygame.display.flip()
    pygame.quit()

if __name__ == '__main__':
    main()
