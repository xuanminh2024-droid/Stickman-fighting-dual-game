mport pygame
from game import Game
from settings import SCREEN_WIDTH, SCREEN_HEIGHT, FPS  # nếu chưa có, hãy tạo settings.py
SCREEN_BG = (30, 30, 30)
def main():R = (50, 160, 255)
    pygame.init()= (255, 80, 80)
    screen = pygame.display.set_mode((SCREEN_WIDTH, SCREEN_HEIGHT))
    pygame.display.set_caption("Street Duel")
    clock = pygame.time.Clock()
    game = Game(screen)
PLAYER_SIZE = (40, 80)
    running = True80)
    while running: 20)
        dt = clock.tick(FPS)  # giới hạn tốc độ, trả về ms since last frame
        events = pygame.event.get()e meaningful with 1000 HP
        for ev in events:
            if ev.type == pygame.QUIT:
                running = False

        game.update(dt, events)  # truyền dt và events
        game.draw()
        pygame.display.flip()
PUNCH_COOLDOWN = 200
    pygame.quit()00
PUNCH_DURATION = 120
if __name__ == "__main__":
    main()
# new skill
SKILL_COOLDOWN = 5000
LASER_DAMAGE = 100
LASER_SPEED = 12
LASER_COLOR = (255, 0, 200)
LASER_SIZE = (6, 4)

class Laser(pygame.sprite.Sprite):
    def __init__(self, pos, direction):
        super().__init__()
        self.image = pygame.Surface(LASER_SIZE)
        self.image.fill(LASER_COLOR)
        self.rect = self.image.get_rect(center=pos)
        self.vx = LASER_SPEED if direction > 0 else -LASER_SPEED
        self.life = 1200  # ms
        self.spawn_time = pygame.time.get_ticks()

    def update(self, dt):
        self.rect.x += int(self.vx)
        if pygame.time.get_ticks() - self.spawn_time > self.life:
            self.kill()
        # remove if off screen
        sw = pygame.display.get_surface().get_width()
        if self.rect.right < 0 or self.rect.left > sw:
            self.kill()

class Player(pygame.sprite.Sprite):
    def __init__(self, pos):
        super().__init__()
        self.width, self.height = PLAYER_SIZE
        self.image = pygame.Surface((self.width, self.height))
        self.image.fill(PLAYER_COLOR)
        self.rect = self.image.get_rect(midbottom=pos)
        self.vx = 0
        self.vy = 0
        self.on_ground = True
        self.facing_right = True
        self.max_hp = 1000
        self.hp = self.max_hp
        self.attacking = False
        self.attack_type = None
        self.last_attack_time = 0
        # skill
        self.last_skill_time = -SKILL_COOLDOWN
        self.has_knife = False

    def can_use_skill(self):
        return pygame.time.get_ticks() - self.last_skill_time >= SKILL_COOLDOWN

    def use_skill(self):
        self.last_skill_time = pygame.time.get_ticks()

    def update(self, dt):
        keys = pygame.key.get_pressed()
        self.vx = 0
        if keys[pygame.K_a]:
            self.vx = -4
            self.facing_right = False
        if keys[pygame.K_d]:
            self.vx = 4
            self.facing_right = True
        if keys[pygame.K_w] and self.on_ground:
            self.vy = -12
            self.on_ground = False

        now = pygame.time.get_ticks()
        # punch (V) faster cooldown/duration
        if keys[pygame.K_v] and now - self.last_attack_time > PUNCH_COOLDOWN:
            self.attacking = True
            self.attack_type = "punch"
            self.last_attack_time = now
            self.attack_duration = PUNCH_DURATION
        # kick (X)
        if keys[pygame.K_x] and now - self.last_attack_time > KICK_COOLDOWN:
            self.attacking = True
            self.attack_type = "kick"
            self.last_attack_time = now
            self.attack_duration = KICK_DURATION

        if self.attacking and now - self.last_attack_time > getattr(self, "attack_duration", KICK_DURATION):
            self.attacking = False
            self.attack_type = None

        # physics
        self.rect.x += self.vx
        self.vy += GRAVITY
        self.rect.y += int(self.vy)
        ground_y = pygame.display.get_surface().get_height() - GROUND_Y_OFFSET
        if self.rect.bottom >= ground_y:
            self.rect.bottom = ground_y
            self.vy = 0
            self.on_ground = True

        if self.attacking:
            self.image.fill(PLAYER_HIT_COLOR)
        else:
            self.image.fill(PLAYER_COLOR)

    def get_attack_rect(self):
        if not self.attacking:
            return None
        w = 24 if self.attack_type == "punch" else 40
        h = 20
        if self.facing_right:
            ax = self.rect.right
        else:
            ax = self.rect.left - w
        ay = self.rect.centery - h // 2
        return pygame.Rect(ax, ay, w, h)


class Enemy(pygame.sprite.Sprite):
    def __init__(self, pos):
        super().__init__()
        self.width, self.height = ENEMY_SIZE
        self.image = pygame.Surface((self.width, self.height))
        self.image.fill(ENEMY_COLOR)
        self.rect = self.image.get_rect(midbottom=pos)
        self.vx = 0
        self.vy = 0
        self.on_ground = True
        self.facing_right = False
        self.max_hp = 1000
        self.hp = self.max_hp
        self.attacking = False
        self.attack_type = None
        self.last_action_time = pygame.time.get_ticks()
        self.next_action_delay = random.randint(600, 1400)
        self.attack_end_timer = None

    def update(self, dt, player_rect=None):
        now = pygame.time.get_ticks()
        if player_rect:
            if abs(self.rect.centerx - player_rect.centerx) > 60:
                self.vx = 2 if player_rect.centerx > self.rect.centerx else -2
                self.facing_right = self.vx > 0
            else:
                self.vx = 0

        self.rect.x += self.vx
        self.vy += GRAVITY
        self.rect.y += int(self.vy)
        ground_y = pygame.display.get_surface().get_height() - GROUND_Y_OFFSET
        if self.rect.bottom >= ground_y:
            self.rect.bottom = ground_y
            self.vy = 0
            self.on_ground = True

        if now - self.last_action_time > self.next_action_delay:
            self.last_action_time = now
            self.next_action_delay = random.randint(700, 1600)
            if random.random() < 0.6:
                self.attacking = True
                self.attack_type = random.choice(["punch", "kick"])
                self.attack_end_timer = now + 220

        if self.attack_end_timer and now > self.attack_end_timer:
            self.finish_attack()
            self.attack_end_timer = None

        if self.attacking:
            self.image.fill((255, 140, 140))
        else:
            self.image.fill(ENEMY_COLOR)

    def finish_attack(self):
        self.attacking = False
        self.attack_type = None

    def get_attack_rect(self):
        if not self.attacking:
            return None
        w = 24 if self.attack_type == "punch" else 40
        h = 20
        if self.facing_right:
            ax = self.rect.right
        else:
            ax = self.rect.left - w
        ay = self.rect.centery - h // 2
        return pygame.Rect(ax, ay, w, h)


class MedKit(pygame.sprite.Sprite):
    def __init__(self, x, top_y=-10):
        super().__init__()
        self.image = pygame.Surface(MEDKIT_SIZE)
        self.image.fill(MEDKIT_COLOR)
        self.rect = self.image.get_rect(midtop=(x, top_y))
        self.vy = 0

    def update(self, dt):
        self.vy += GRAVITY * MEDKIT_FALL_MULTIPLIER
        self.rect.y += int(self.vy)
        screen_h = pygame.display.get_surface().get_height()
        if self.rect.top > screen_h:
            self.kill()


class Game:
    def __init__(self, screen):
        self.screen = screen
        self.screen_rect = screen.get_rect()
        self.bg_color = SCREEN_BG
        self.all_sprites = pygame.sprite.Group()
        ground_y = self.screen_rect.height - GROUND_Y_OFFSET
        self.player = Player((100, ground_y))
        self.enemy = Enemy((self.screen_rect.width - 100, ground_y))
        self.all_sprites.add(self.player, self.enemy)
        self.font = pygame.font.SysFont(None, 24)

        self.items = pygame.sprite.Group()
        self.last_medkit_time = pygame.time.get_ticks()
        self.next_medkit_delay = random.randint(5000, 12000)

        self.projectiles = pygame.sprite.Group()

        # coins for shop
        self.coins = 0

        # state: 'running', 'gameover', 'shop'
        self.state = "running"

    def spawn_medkit(self):
        x = random.randint(40, self.screen_rect.width - 40)
        med = MedKit(x)
        self.items.add(med)
        self.all_sprites.add(med)

    def spawn_laser(self, direction):
        # spawn from player's center slightly ahead
        pos = (self.player.rect.centerx + (self.player.width//2 + 6) * (1 if direction>0 else -1),
               self.player.rect.centery)
        l = Laser(pos, direction)
        self.projectiles.add(l)
        self.all_sprites.add(l)

    def restart(self):
        ground_y = self.screen_rect.height - GROUND_Y_OFFSET
        self.player.rect.midbottom = (100, ground_y)
        self.player.hp = self.player.max_hp
        self.player.has_knife = False
        self.enemy.rect.midbottom = (self.screen_rect.width - 100, ground_y)
        self.enemy.hp = self.enemy.max_hp
        self.items.empty()
        self.projectiles.empty()
        self.all_sprites = pygame.sprite.Group(self.player, self.enemy)
        self.state = "running"

    def update(self, dt, events):
        # xử lý events được truyền từ main (không gọi pygame.event.get() ở đây)
        for ev in events:
            if ev.type == pygame.QUIT:
                pass
            if self.state == "gameover":
                if ev.type == pygame.KEYDOWN:
                    if ev.key == pygame.K_r:
                        self.restart()
                    if ev.key == pygame.K_o:
                        self.state = "shop"
            elif self.state == "shop":
                if ev.type == pygame.KEYDOWN:
                    if ev.key == pygame.K_1:  # Revive cost 50
                        if self.coins >= 50:
                            self.coins -= 50
                            self.player.hp = 50
                            self.state = "running"
                    if ev.key == pygame.K_2:  # Full heal cost 30
                        if self.coins >= 30:
                            self.coins -= 30
                            self.player.hp = self.player.max_hp
                            self.state = "running"
                    if ev.key == pygame.K_3:  # Buy Knife (equip)
                        if self.coins >= 40:
                            self.coins -= 40
                            self.player.has_knife = True
                    if ev.key == pygame.K_r:
                        self.restart()
            else:
                # in running state listen for skill key
                if ev.type == pygame.KEYDOWN:
                    if ev.key == pygame.K_SPACE and self.player.can_use_skill():
                        # use skill: spawn laser toward facing direction
                        dir = 1 if self.player.facing_right else -1
                        self.spawn_laser(dir)
                        self.player.use_skill()

        if self.state != "running":
            return

        # dùng dt thay vì tạo Clock ở đây
        self.player.update(dt)
        self.enemy.update(dt, player_rect=self.player.rect)
        self.items.update(dt)
        self.projectiles.update(dt)

        # spawn medkit occasionally
        now = pygame.time.get_ticks()
        if now - self.last_medkit_time > self.next_medkit_delay:
            self.last_medkit_time = now
            self.next_medkit_delay = random.randint(8000, 15000)
            self.spawn_medkit()

        # player attack hits enemy (limit to attack start)
        pr = self.player.get_attack_rect()
        now = pygame.time.get_ticks()
        if pr and pr.colliderect(self.enemy.rect) and now - self.player.last_attack_time < 80:
            # damage: if player has knife, punch -> slash 20
            if self.player.attack_type == "punch":
                damage = 20 if self.player.has_knife else 10
            else:
                damage = 12  # kick
            self.enemy.hp = max(0, self.enemy.hp - damage)
            self.coins += 10
            if self.player.facing_right:
                self.enemy.rect.x += 10
            else:
                self.enemy.rect.x -= 10

        # enemy attack hits player (limit to attack start)
        er = self.enemy.get_attack_rect()
        if er and er.colliderect(self.player.rect) and self.enemy.attack_end_timer and pygame.time.get_ticks() - (self.enemy.attack_end_timer - 220) < 80:
            self.player.hp = max(0, self.player.hp - (10 if self.enemy.attack_type == "kick" else 6))

        # laser collisions with enemy
        for laser in self.projectiles:
            if laser.rect.colliderect(self.enemy.rect):
                self.enemy.hp = max(0, self.enemy.hp - LASER_DAMAGE)
                laser.kill()
                self.coins += 20

        # medkit pickup
        picked = pygame.sprite.spritecollide(self.player, self.items, dokill=True)
        if picked:
            for _ in picked:
                self.player.hp = min(self.player.max_hp, self.player.hp + MEDKIT_HEAL)

        # check death -> game over
        if self.player.hp <= 0:
            self.state = "gameover"

        # enemy death -> award coins and respawn enemy
        if self.enemy.hp <= 0:
            self.coins += 50
            self.enemy.hp = self.enemy.max_hp
            ground_y = self.screen_rect.height - GROUND_Y_OFFSET
            self.enemy.rect.midbottom = (self.screen_rect.width - 100, ground_y)

    def draw(self):
        self.screen.fill(self.bg_color)
        ground_y = self.screen_rect.height - GROUND_Y_OFFSET
        pygame.draw.line(self.screen, (80, 80, 80), (0, ground_y), (self.screen_rect.width, ground_y), 4)

        self.all_sprites.draw(self.screen)

        ar = self.player.get_attack_rect()
        if ar:
            pygame.draw.rect(self.screen, (255, 200, 0), ar)
        er = self.enemy.get_attack_rect()
        if er:
            pygame.draw.rect(self.screen, (255, 200, 50), er)

        # HUD
        self._draw_health_bar(self.player.hp, self.player.max_hp, 20, 20, 300, 20)
        self._draw_health_bar(self.enemy.hp, self.enemy.max_hp, self.screen_rect.width - 320, 20, 300, 20)
        coin_txt = self.font.render(f"Coins: {self.coins}", True, (255, 215, 0))
        self.screen.blit(coin_txt, (20, 50))

        # weapon / skill status
        weapon_text = "Knife (equipped)" if self.player.has_knife else "Fist"
        self.screen.blit(self.font.render(f"Weapon: {weapon_text}", True, (255,255,255)), (20, 80))
        cd = max(0, SKILL_COOLDOWN - (pygame.time.get_ticks() - self.player.last_skill_time))
        cd_s = f"{cd//1000}.{(cd%1000)//100}s" if cd>0 else "Ready"
        self.screen.blit(self.font.render(f"Skill (SPACE): Laser - {cd_s}", True, (255,255,255)), (20, 100))

        if self.state == "gameover":
            self._draw_overlay("GAME OVER - Press R to Restart or O for Shop")
        elif self.state == "shop":
            lines = [
                "SHOP - press number to buy:",
                "1) Revive (50 coins) - revive to 50 HP",
                "2) Full Heal (30 coins)",
                "3) Knife (40 coins) - punch becomes slash (20 dmg)",
                "R) Restart"
            ]
            self._draw_shop(lines)

    def _draw_health_bar(self, hp, max_hp, x, y, w, h):
        pygame.draw.rect(self.screen, HEALTH_BG, (x, y, w, h))
        pct = max(0, hp) / max_hp
        pygame.draw.rect(self.screen, HEALTH_FG, (x, y, int(w * pct), h))
        txt = self.font.render(f"{hp}/{max_hp}", True, (255,255,255))
        self.screen.blit(txt, (x + w//2 - txt.get_width()//2, y + h//2 - txt.get_height()//2))

    def _draw_overlay(self, text):
        s = pygame.Surface((self.screen_rect.width, self.screen_rect.height), pygame.SRCALPHA)
        s.fill((0,0,0,180))
        self.screen.blit(s, (0,0))
        txt = self.font.render(text, True, (255,255,255))
        self.screen.blit(txt, (self.screen_rect.width//2 - txt.get_width()//2, self.screen_rect.height//2 - 10))

    def _draw_shop(self, lines):
        s = pygame.Surface((self.screen_rect.width, self.screen_rect.height), pygame.SRCALPHA)
        s.fill((0,0,0,200))
        self.screen.blit(s, (0,0))
        y = self.screen_rect.height//2 - 80
        for line in lines:
            txt = self.font.render(line, True, (255,255,255))
            self.screen.blit(txt, (self.screen_rect.width//2 - txt.get_width()//2, y))
            y += 30
