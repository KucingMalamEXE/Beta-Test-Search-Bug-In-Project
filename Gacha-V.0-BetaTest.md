import pygame
import random
import sys
import time
import math

# Inisialisasi Pygame
pygame.init()

# Konfigurasi layar
WIDTH, HEIGHT = 800, 600
screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("Gacha System Demo")

# Warna
BACKGROUND = (20, 20, 35)
WHITE = (255, 255, 255)
BLACK = (10, 10, 20)
GOLD = (255, 215, 0)
PURPLE = (180, 70, 255)
BLUE = (70, 130, 255)
GREEN = (100, 220, 100)
RED = (255, 80, 80)
DARK_PURPLE = (80, 0, 120)
HOVER_PURPLE = (160, 50, 200)
YELLOW = (255, 255, 100)

# Font
try:
    title_font = pygame.font.SysFont("Arial", 48, bold=True)
    item_font = pygame.font.SysFont("Arial", 28, bold=True)
    button_font = pygame.font.SysFont("Arial", 32, bold=True)
    info_font = pygame.font.SysFont("Arial", 22)
    rate_font = pygame.font.SysFont("Arial", 20)
except:
    # Fallback jika font tidak tersedia
    title_font = pygame.font.Font(None, 48)
    item_font = pygame.font.Font(None, 28)
    button_font = pygame.font.Font(None, 32)
    info_font = pygame.font.Font(None, 22)
    rate_font = pygame.font.Font(None, 20)

# Definisi raritas dengan rates yang lebih realistis
RARITIES = [
    {"name": "Common", "rate": 70.0, "color": GREEN, "weight": 70.0},
    {"name": "Uncommon", "rate": 20.0, "color": BLUE, "weight": 20.0},
    {"name": "Rare", "rate": 7.0, "color": PURPLE, "weight": 7.0},
    {"name": "Epic", "rate": 2.9, "color": GOLD, "weight": 2.9},
    {"name": "Legendary", "rate": 0.1, "color": RED, "weight": 0.1}
]

# Total rate harus 100%
total_rate = sum(rarity["rate"] for rarity in RARITIES)
if abs(total_rate - 100.0) > 0.001:
    print(f"Warning: Total rates = {total_rate}%, should be 100%")

# Generate item pool
ITEMS = []
item_id = 1

# Distribusi item yang lebih proporsional
rarity_distribution = {
    "Common": 25,    # 25 items
    "Uncommon": 15,  # 15 items
    "Rare": 10,      # 10 items
    "Epic": 4,       # 4 items
    "Legendary": 1   # 1 item
}

for rarity_name, count in rarity_distribution.items():
    rarity_info = next(r for r in RARITIES if r["name"] == rarity_name)
    for j in range(count):
        ITEMS.append({
            "id": item_id,
            "name": f"Item {item_id:03d}",
            "rarity": rarity_name,
            "color": rarity_info["color"],
            "rate": rarity_info["rate"],
            "weight": rarity_info["weight"]
        })
        item_id += 1

class Button:
    def __init__(self, x, y, width, height, text, color, hover_color):
        self.rect = pygame.Rect(x, y, width, height)
        self.text = text
        self.color = color
        self.hover_color = hover_color
        self.is_hovered = False
        
    def draw(self, surface):
        # Draw button
        color = self.hover_color if self.is_hovered else self.color
        pygame.draw.rect(surface, color, self.rect, border_radius=10)
        
        # Border
        border_color = GOLD if self.is_hovered else WHITE
        pygame.draw.rect(surface, border_color, self.rect, 3, border_radius=10)
        
        # Text dengan shadow untuk readability
        text_surf = button_font.render(self.text, True, WHITE)
        text_rect = text_surf.get_rect(center=self.rect.center)
        
        # Shadow
        shadow_surf = button_font.render(self.text, True, (0, 0, 0, 128))
        shadow_rect = shadow_surf.get_rect(center=(self.rect.centerx + 2, self.rect.centery + 2))
        surface.blit(shadow_surf, shadow_rect)
        
        surface.blit(text_surf, text_rect)
        
    def update(self, mouse_pos):
        self.is_hovered = self.rect.collidepoint(mouse_pos)
        
    def is_clicked(self, event):
        if event.type == pygame.MOUSEBUTTONDOWN and event.button == 1:
            return self.rect.collidepoint(event.pos)
        return False

class Particle:
    def __init__(self, x, y, color, size=3, speed=2):
        self.x = x
        self.y = y
        self.color = color
        self.size = random.randint(size-1, size+2)
        angle = random.uniform(0, 2 * math.pi)
        self.speed_x = math.cos(angle) * speed * random.uniform(0.5, 1.5)
        self.speed_y = math.sin(angle) * speed * random.uniform(0.5, 1.5)
        self.life = random.randint(20, 40)
        self.max_life = self.life
        self.gravity = 0.1
        
    def update(self):
        self.x += self.speed_x
        self.y += self.speed_y
        self.speed_y += self.gravity
        self.life -= 1
        return self.life > 0
        
    def draw(self, surface):
        alpha = int(255 * (self.life / self.max_life))
        r, g, b = self.color
        
        # Create surface with alpha
        particle_surf = pygame.Surface((self.size * 2, self.size * 2), pygame.SRCALPHA)
        pygame.draw.circle(particle_surf, (r, g, b, alpha), 
                          (self.size, self.size), self.size)
        surface.blit(particle_surf, (int(self.x - self.size), int(self.y - self.size)))

class GachaSystem:
    def __init__(self):
        self.particles = []
        self.last_result = None
        self.show_result = False
        self.animation_start = 0
        self.pull_count = 0
        self.pull_history = []
        self.rarity_counts = {rarity["name"]: 0 for rarity in RARITIES}
        
    def perform_gacha(self):
        """Perform a single gacha pull with proper weighted random"""
        self.pull_count += 1
        
        # Weighted random selection menggunakan weights dari RARITIES
        total_weight = sum(rarity["weight"] for rarity in RARITIES)
        rand = random.uniform(0, total_weight)
        cumulative = 0
        
        selected_rarity = None
        for rarity in RARITIES:
            cumulative += rarity["weight"]
            if rand <= cumulative:
                selected_rarity = rarity["name"]
                break
        
        if not selected_rarity:  # Fallback jika ada masalah
            selected_rarity = "Common"
        
        # Update rarity counter
        self.rarity_counts[selected_rarity] += 1
        
        # Get all items of selected rarity
        available_items = [item for item in ITEMS if item["rarity"] == selected_rarity]
        result = random.choice(available_items)
        
        # Add to history (keep last 10)
        self.pull_history.append(result.copy())  # Gunakan copy agar tidak reference
        if len(self.pull_history) > 10:
            self.pull_history.pop(0)
        
        return result
    
    def create_explosion(self, x, y, color, count=30):
        """Create particle explosion effect"""
        for _ in range(count):
            self.particles.append(Particle(x, y, color))
            
    def create_sparkles(self, rect, color, count=8):
        """Create sparkles around a rectangle"""
        for _ in range(count):
            x = random.randint(rect.left, rect.right)
            y = random.randint(rect.top, rect.bottom)
            self.particles.append(Particle(x, y, color, size=2, speed=1))
    
    def update_particles(self):
        """Update all particles"""
        self.particles = [p for p in self.particles if p.update()]
    
    def draw_particles(self, surface):
        """Draw all particles"""
        for particle in self.particles:
            particle.draw(surface)
    
    def draw_result(self, surface, result):
        """Draw the gacha result"""
        # Semi-transparent overlay
        overlay = pygame.Surface((WIDTH, HEIGHT), pygame.SRCALPHA)
        overlay.fill((0, 0, 0, 180))
        surface.blit(overlay, (0, 0))
        
        # Result box dengan ukuran dinamis
        box_width, box_height = 400, 280
        box_x, box_y = WIDTH//2 - box_width//2, HEIGHT//2 - box_height//2
        result_rect = pygame.Rect(box_x, box_y, box_width, box_height)
        
        # Glow effect untuk item rare ke atas
        if result["rarity"] in ["Rare", "Epic", "Legendary"]:
            glow_size = 30
            for i in range(glow_size, 0, -2):
                alpha = int(150 * (i / glow_size))
                glow_rect = result_rect.inflate(i*2, i*2)
                glow_surf = pygame.Surface(glow_rect.size, pygame.SRCALPHA)
                pygame.draw.rect(glow_surf, (*result["color"], alpha), 
                                glow_surf.get_rect(), border_radius=20)
                surface.blit(glow_surf, glow_rect)
        
        # Main box dengan gradient effect
        pygame.draw.rect(surface, (25, 25, 40), result_rect, border_radius=15)
        pygame.draw.rect(surface, result["color"], result_rect, 4, border_radius=15)
        
        # Header dengan rarity name
        header_rect = pygame.Rect(box_x, box_y, box_width, 50)
        pygame.draw.rect(surface, result["color"], header_rect, border_radius=15)
        
        rarity_text = button_font.render(result["rarity"], True, WHITE)
        surface.blit(rarity_text, (WIDTH//2 - rarity_text.get_width()//2, box_y + 10))
        
        # Item name
        name_text = item_font.render(result["name"], True, WHITE)
        surface.blit(name_text, (WIDTH//2 - name_text.get_width()//2, box_y + 70))
        
        # ID dan Rate
        id_text = info_font.render(f"ID: #{result['id']:03d}", True, YELLOW)
        surface.blit(id_text, (WIDTH//2 - id_text.get_width()//2, box_y + 120))
        
        # Format rate display
        rate_value = result["rate"]
        rate_str = f"Drop Rate: {rate_value:.1f}%" if rate_value >= 1 else f"Drop Rate: {rate_value:.3f}%"
        
        rate_text = info_font.render(rate_str, True, WHITE)
        surface.blit(rate_text, (WIDTH//2 - rate_text.get_width()//2, box_y + 150))
        
        # Pull counter
        pull_text = info_font.render(f"Pull #{self.pull_count}", True, (200, 200, 200))
        surface.blit(pull_text, (WIDTH//2 - pull_text.get_width()//2, box_y + 180))
        
        # Continue instruction
        continue_text = rate_font.render("Click anywhere or press ESC to continue", True, (180, 180, 180))
        surface.blit(continue_text, (WIDTH//2 - continue_text.get_width()//2, box_y + 230))

def draw_stats(surface, gacha_system):
    """Draw pull statistics"""
    stats_y = 140
    
    # Total pulls dengan box background
    stats_bg = pygame.Rect(20, stats_y - 10, 250, 180)
    pygame.draw.rect(surface, (30, 30, 45), stats_bg, border_radius=10)
    pygame.draw.rect(surface, BLUE, stats_bg, 2, border_radius=10)
    
    # Total pulls
    pulls_text = info_font.render(f"Total Pulls: {gacha_system.pull_count}", True, WHITE)
    surface.blit(pulls_text, (30, stats_y))
    
    # Rarity distribution
    dist_y = stats_y + 30
    dist_text = info_font.render("Rarity Distribution:", True, WHITE)
    surface.blit(dist_text, (30, dist_y))
    
    for i, rarity in enumerate(RARITIES):
        count = gacha_system.rarity_counts[rarity["name"]]
        percentage = (count / gacha_system.pull_count * 100) if gacha_system.pull_count > 0 else 0
        dist_item = rate_font.render(
            f"{rarity['name'][0]}: {count} ({percentage:.1f}%)", 
            True, rarity["color"]
        )
        surface.blit(dist_item, (40, dist_y + 25 + i * 22))
    
    # Last pulls
    if gacha_system.pull_history:
        history_y = dist_y + 25 + len(RARITIES) * 22 + 10
        history_text = info_font.render("Recent Pulls:", True, WHITE)
        surface.blit(history_text, (30, history_y))
        
        for i, item in enumerate(reversed(gacha_system.pull_history[-5:])):
            item_text = rate_font.render(
                f"{item['name']} ({item['rarity'][0]})", 
                True, item['color']
            )
            surface.blit(item_text, (40, history_y + 25 + i * 22))

def draw_stars_background(surface):
    """Draw animated star background"""
    for i in range(100):
        x = (i * 13) % WIDTH
        y = (i * 7 + pygame.time.get_ticks() // 100) % HEIGHT
        size = random.choice([1, 1, 2])  # Mostly small stars
        brightness = random.randint(150, 255)
        pygame.draw.circle(surface, (brightness, brightness, brightness, 100), 
                          (int(x), int(y)), size)

def main():
    # Initialize systems
    gacha = GachaSystem()
    gacha_button = Button(WIDTH//2 - 120, HEIGHT - 100, 240, 60, 
                         "PULL GACHA", DARK_PURPLE, HOVER_PURPLE)
    
    clock = pygame.time.Clock()
    running = True
    
    while running:
        current_time = time.time()
        mouse_pos = pygame.mouse.get_pos()
        
        # Event handling
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                running = False
                
            if event.type == pygame.KEYDOWN:
                if event.key == pygame.K_ESCAPE:
                    if gacha.show_result:
                        gacha.show_result = False
                    else:
                        running = False
                elif event.key == pygame.K_SPACE and not gacha.show_result:
                    # Spacebar for quick pull
                    gacha.last_result = gacha.perform_gacha()
                    gacha.show_result = True
                    gacha.animation_start = current_time
                    gacha.create_explosion(WIDTH//2, HEIGHT//2, gacha.last_result["color"], 40)
            
            if gacha_button.is_clicked(event) and not gacha.show_result:
                gacha.last_result = gacha.perform_gacha()
                gacha.show_result = True
                gacha.animation_start = current_time
                gacha.create_explosion(WIDTH//2, HEIGHT//2, gacha.last_result["color"], 40)
                
            elif event.type == pygame.MOUSEBUTTONDOWN and gacha.show_result:
                gacha.show_result = False
        
        # Update
        gacha_button.update(mouse_pos)
        gacha.update_particles()
        
        # Create sparkles for rare+ items during animation
        if gacha.show_result and gacha.last_result:
            if gacha.last_result["rarity"] in ["Rare", "Epic", "Legendary"]:
                if current_time - gacha.animation_start < 3:
                    result_rect = pygame.Rect(WIDTH//2 - 200, HEIGHT//2 - 140, 400, 280)
                    gacha.create_sparkles(result_rect, gacha.last_result["color"], 3)
        
        # Draw
        screen.fill(BACKGROUND)
        
        # Draw animated stars
        draw_stars_background(screen)
        
        # Title dengan efek glow
        title_text = title_font.render("GACHA SYSTEM", True, GOLD)
        title_shadow = title_font.render("GACHA SYSTEM", True, (100, 80, 0, 100))
        screen.blit(title_shadow, (WIDTH//2 - title_text.get_width()//2 + 4, 34))
        screen.blit(title_text, (WIDTH//2 - title_text.get_width()//2, 30))
        
        # Rates info
        rates_y = 85
        rates_bg = pygame.Rect(WIDTH//2 - 300, rates_y - 5, 600, 30)
        pygame.draw.rect(screen, (40, 40, 60), rates_bg, border_radius=5)
        
        rates_text = []
        for rarity in RARITIES:
            rate_str = f"{rarity['rate']:.1f}%" if rarity["rate"] >= 1 else f"{rarity['rate']:.3f}%"
            rates_text.append(f"{rarity['name']}: {rate_str}")
        
        rates_display = "  |  ".join(rates_text)
        rates_surface = rate_font.render(rates_display, True, WHITE)
        screen.blit(rates_surface, (WIDTH//2 - rates_surface.get_width()//2, rates_y))
        
        # Item pool info
        pool_text = info_font.render(f"Item Pool: {len(ITEMS)} items total", True, (200, 200, 220))
        screen.blit(pool_text, (WIDTH//2 - pool_text.get_width()//2, rates_y + 30))
        
        # Draw stats
        draw_stats(screen, gacha)
        
        # Draw button
        gacha_button.draw(screen)
        
        # Draw particles
        gacha.draw_particles(screen)
        
        # Draw result if showing
        if gacha.show_result and gacha.last_result:
            gacha.draw_result(screen, gacha.last_result)
            
            # Special effects for epic and legendary
            if gacha.last_result["rarity"] in ["Epic", "Legendary"]:
                elapsed = current_time - gacha.animation_start
                if elapsed < 2:
                    # Pulsing glow effect
                    pulse = abs(math.sin(elapsed * 5)) * 100
                    glow_surf = pygame.Surface((WIDTH, HEIGHT), pygame.SRCALPHA)
                    glow_surf.fill((*gacha.last_result["color"], int(pulse)))
                    screen.blit(glow_surf, (0, 0))
        
        # Instructions
        if not gacha.show_result:
            instr_text = rate_font.render("Press SPACE for quick pull | ESC to exit", True, (180, 180, 180))
            screen.blit(instr_text, (WIDTH//2 - instr_text.get_width()//2, HEIGHT - 40))
        else:
            # Show rarity during pull
            if gacha.last_result:
                rarity_effect = info_font.render(
                    f"You got a {gacha.last_result['rarity']} item!", 
                    True, gacha.last_result["color"]
                )
                screen.blit(rarity_effect, (WIDTH//2 - rarity_effect.get_width()//2, 50))
        
        # FPS counter (uncomment for debugging)
        # fps_text = rate_font.render(f"FPS: {int(clock.get_fps())}", True, WHITE)
        # screen.blit(fps_text, (WIDTH - 80, 20))
        
        pygame.display.flip()
        clock.tick(60)
    
    pygame.quit()
    sys.exit()

if __name__ == "__main__":
    main()
