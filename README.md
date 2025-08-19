import pygame
import math
import random
from enum import Enum

# Initialize Pygame
pygame.init()

# Constants
WIN_WIDTH = 1000
WIN_HEIGHT = 700
GROUND_HEIGHT = 50
FPS = 60
DT = 1 / FPS
GRAVITY = 300  # Gravitational acceleration (pixels/s^2)
ELASTICITY = 0.6  # Coefficient of restitution
FRICTION = 0.98  # Friction coefficient
MAX_ELASTIC_LENGTH = 150  # Maximum stretch length
SLINGSHOT_X = 150
SLINGSHOT_Y = WIN_HEIGHT - 125

# Colors
WHITE = (255, 255, 255)
BLACK = (0, 0, 0)
RED = (255, 0, 0)
GREEN = (0, 255, 0)
BLUE = (0, 0, 255)
YELLOW = (255, 255, 0)
BROWN = (139, 69, 19)
PURPLE = (128, 0, 128)
LIGHTBLUE = (173, 216, 230)
GRASS_COLOR = (190, 217, 60)
GROUND_COLOR = (89, 66, 51)

class Ball:
    def __init__(self, x, y, radius, mass, color, loaded=False):
        self.x = x
        self.y = y
        self.radius = radius
        self.mass = mass
        self.color = color
        self.vx = 0
        self.vy = 0
        self.loaded = loaded
        self.visible = True

    def move(self):
        if self.loaded or not self.visible:
            return
        
        # Update velocity due to gravity (acts downward, increasing y)
        self.vy += GRAVITY * DT
        
        # Update position
        self.x += self.vx * DT
        self.y += self.vy * DT

        # Apply friction only when on ground
        if self.y + self.radius >= WIN_HEIGHT - GROUND_HEIGHT:
            self.vx *= FRICTION
            
            # Handle ground bounce
            if self.vy > 0:  # Moving downward
                self.vy *= -ELASTICITY
                self.y = WIN_HEIGHT - GROUND_HEIGHT - self.radius
                
                # Stop tiny bounces
                if abs(self.vy) < 50:
                    self.vy = 0

        # Wall collisions
        if self.x - self.radius < 0:
            self.vx = abs(self.vx) * ELASTICITY
            self.x = self.radius
        elif self.x + self.radius > WIN_WIDTH:
            self.vx = -abs(self.vx) * ELASTICITY
            self.x = WIN_WIDTH - self.radius

    def draw(self, screen):
        if self.visible:
            pygame.draw.circle(screen, self.color, (int(self.x), int(self.y)), self.radius)
            pygame.draw.circle(screen, BLACK, (int(self.x), int(self.y)), self.radius, 2)

    def set_velocity(self, speed, angle):
        """Launch the ball with an initial speed and angle (degrees)."""
        angle_rad = math.radians(angle)
        self.vx = speed * math.cos(angle_rad)
        self.vy = speed * math.sin(angle_rad)
        self.loaded = False

    def is_dead(self):
        """Check if the ball has stopped moving."""
        on_ground = self.y + self.radius >= WIN_HEIGHT - GROUND_HEIGHT - 5
        nearly_stopped = abs(self.vx) < 10 and abs(self.vy) < 10
        return (on_ground and nearly_stopped) and not self.loaded

class Block:
    def __init__(self, x, y, width, height, mass, color, max_health=100):
        self.x = x
        self.y = y
        self.width = width
        self.height = height
        self.mass = mass
        self.color = color
        self.original_color = color
        self.vx = 0
        self.vy = 0
        self.max_health = max_health
        self.health = max_health
        self.visible = True
        self.angle = 0  # Angle in degrees
        self.angular_velocity = 0  # In degrees per second
        # Moment of inertia for a rectangle
        self.moment_of_inertia = (1/12) * self.mass * (self.width**2 + self.height**2)
        self.cracked = False
        
    def take_damage(self, damage_amount):
        """Apply damage to the block"""
        self.health -= damage_amount
        
        # Check if block should show cracks (health below 50%)
        if self.health <= self.max_health * 0.5 and not self.cracked:
            self.cracked = True
            # Darken the color to show damage
            r, g, b = self.original_color
            self.color = (max(0, r - 50), max(0, g - 50), max(0, b - 50))
        
        # Check if block is destroyed
        if self.health <= 0:
            self.visible = False
            return True  # Block destroyed
        return False  # Block still alive

    def apply_impulse(self, impulse_x, impulse_y, point_of_impact):
        """Apply an impulse to the block, resulting in linear and angular velocity changes."""
        # Linear motion
        self.vx += impulse_x / self.mass
        self.vy += impulse_y / self.mass

        # Rotational motion (Torque)
        center_of_mass_x = self.x + self.width / 2
        center_of_mass_y = self.y + self.height / 2

        # r is the vector from the center of mass to the point of impact
        r_x = point_of_impact[0] - center_of_mass_x
        r_y = point_of_impact[1] - center_of_mass_y

        # Torque is the cross product of r and the impulse (J)
        # For 2D, this simplifies to r_x * J_y - r_y * J_x
        torque = r_x * impulse_y - r_y * impulse_x

        # Angular acceleration = Torque / Moment of Inertia
        angular_acceleration = torque / self.moment_of_inertia

        # Change in angular velocity
        self.angular_velocity += math.degrees(angular_acceleration)

    def get_rotated_corners(self):
        """Get the corners of the rotated rectangle"""
        # Half dimensions
        hw = self.width / 2
        hh = self.height / 2
        
        # Center position
        cx = self.x + hw
        cy = self.y + hh
        
        # Calculate rotated corners
        rad = math.radians(self.angle)
        cos_r = math.cos(rad)
        sin_r = math.sin(rad)
        
        corners = []
        for dx, dy in [(-hw, -hh), (hw, -hh), (hw, hh), (-hw, hh)]:
            # Rotate point around center
            rx = dx * cos_r - dy * sin_r
            ry = dx * sin_r + dy * cos_r
            corners.append((cx + rx, cy + ry))
        
        return corners

    def move(self):
        if not self.visible:
            return
            
        # Update velocity due to gravity
        self.vy += GRAVITY * DT

        # Update position
        self.x += self.vx * DT
        self.y += self.vy * DT
        
        # Update rotation
        self.angle += self.angular_velocity * DT
        
        # Apply angular damping
        self.angular_velocity *= 0.99
        
        # Get rotated corners for ground collision
        corners = self.get_rotated_corners()
        lowest_point = max(corner[1] for corner in corners) if corners else self.y + self.height
        
        # Ground collision
        if lowest_point >= WIN_HEIGHT - GROUND_HEIGHT:
            # Find how much we're below ground
            penetration = lowest_point - (WIN_HEIGHT - GROUND_HEIGHT)
            self.y -= penetration
            
            # Bounce
            if self.vy > 0:
                self.vy *= -ELASTICITY * 0.5
                self.vx *= FRICTION
                self.angular_velocity *= 0.7  # Reduce spin on ground hit
                
                # Stop tiny movements
                if abs(self.vy) < 20:
                    self.vy = 0
                if abs(self.angular_velocity) < 0.1:
                    self.angular_velocity *= 0.9

        # Wall collisions (simplified - using center point)
        center_x = self.x + self.width / 2
        if center_x - self.width / 2 < 0:
            self.x = -self.width / 2 + (self.width / 2)
            self.vx = abs(self.vx) * ELASTICITY
            self.angular_velocity *= -0.5
        elif center_x + self.width / 2 > WIN_WIDTH:
            self.x = WIN_WIDTH - self.width / 2 - (self.width / 2)
            self.vx = -abs(self.vx) * ELASTICITY
            self.angular_velocity *= -0.5

    def draw(self, screen):
        if self.visible:
            # Create a surface for the block
            block_surface = pygame.Surface((self.width, self.height), pygame.SRCALPHA)
            block_surface.fill(self.color)
            
            # Draw cracks if damaged
            if self.cracked:
                crack_color = (50, 50, 50)
                # Draw diagonal cracks
                pygame.draw.line(block_surface, crack_color, 
                               (0, self.height//3), 
                               (self.width//2, self.height//2), 2)
                pygame.draw.line(block_surface, crack_color, 
                               (self.width, self.height//3), 
                               (self.width//2, self.height//2), 2)
                pygame.draw.line(block_surface, crack_color, 
                               (self.width//3, 0), 
                               (self.width//2, self.height//2), 2)
                pygame.draw.line(block_surface, crack_color, 
                               (2*self.width//3, self.height), 
                               (self.width//2, self.height//2), 2)
            
            # Add border
            pygame.draw.rect(block_surface, BLACK, block_surface.get_rect(), 2)
            
            # Rotate the surface
            rotated_surface = pygame.transform.rotate(block_surface, self.angle)
            
            # Get the rect and center it on the block's position
            rotated_rect = rotated_surface.get_rect(center = (self.x + self.width/2, self.y + self.height/2))
            
            # Draw the rotated block
            screen.blit(rotated_surface, rotated_rect)
            
            # Draw health bar above block if damaged
            if self.health < self.max_health:
                bar_width = 30
                bar_height = 4
                bar_x = self.x + self.width/2 - bar_width / 2
                bar_y = self.y - 10
                
                # Background (red)
                pygame.draw.rect(screen, RED, 
                               (bar_x, bar_y, bar_width, bar_height))
                
                # Health (green)
                health_percentage = self.health / self.max_health
                pygame.draw.rect(screen, GREEN, 
                               (bar_x, bar_y, bar_width * health_percentage, bar_height))
                
                # Border
                pygame.draw.rect(screen, BLACK, 
                               (bar_x, bar_y, bar_width, bar_height), 1)

    def damage(self, ball_color):
        """Legacy method - now uses take_damage internally"""
        if ball_color == RED:
            return self.take_damage(self.max_health)  # Full damage
        else:
            return self.take_damage(self.max_health * 0.3)  # 30% damage

class Slingshot:
    def __init__(self):
        self.loaded_pos = None
        self.object = None
        self.anchor_left = (135, SLINGSHOT_Y)
        self.anchor_right = (165, SLINGSHOT_Y)
        self.base_pos = (SLINGSHOT_X, WIN_HEIGHT - 100)
        self.trajectory_points = []

    def draw(self, screen):
        # Draw base
        pygame.draw.rect(screen, BROWN, (145, WIN_HEIGHT - 100, 10, 60))
        
        # Draw arms
        pygame.draw.line(screen, BROWN, (150, WIN_HEIGHT - 100), (125, WIN_HEIGHT - 150), 5)
        pygame.draw.line(screen, BROWN, (150, WIN_HEIGHT - 100), (175, WIN_HEIGHT - 150), 5)
        
        # Draw trajectory preview
        for i, point in enumerate(self.trajectory_points):
            if i % 3 == 0:  # Draw every 3rd point for dotted effect
                pygame.draw.circle(screen, (255, 255, 255, 128), point, 3)
        
        # Draw elastic bands if loaded
        if self.loaded_pos and self.object:
            pygame.draw.line(screen, BLACK, self.anchor_left, self.loaded_pos, 3)
            pygame.draw.line(screen, BLACK, self.anchor_right, self.loaded_pos, 3)

    def calculate_trajectory(self):
        """Calculate trajectory preview points"""
        self.trajectory_points = []
        if not self.loaded_pos or not self.object:
            return
        
        # Calculate pull vector (from pulled position to slingshot center)
        pull_dx = SLINGSHOT_X - self.loaded_pos[0]
        pull_dy = SLINGSHOT_Y - self.loaded_pos[1]
        stretch_length = math.hypot(pull_dx, pull_dy)
        
        if stretch_length == 0:
            return
        
        # Calculate initial velocity (in direction of pull vector)
        velocity = min(stretch_length * 6, 800)
        angle = math.atan2(-pull_dy, pull_dx)
        
        vx = velocity * math.cos(angle)
        vy = velocity * math.sin(angle)
        
        # Simulate trajectory starting from current position
        x, y = self.loaded_pos[0], self.loaded_pos[1]
        dt = 0.05
        
        for _ in range(30):  # Show 30 points
            x += vx * dt
            y += vy * dt
            vy += GRAVITY * dt  # Apply gravity
            
            # Stop if we hit boundaries
            if y > WIN_HEIGHT - GROUND_HEIGHT or x < 0 or x > WIN_WIDTH:
                break
                
            self.trajectory_points.append((int(x), int(y)))

    def attach_object(self, obj):
        self.object = obj
        obj.x = SLINGSHOT_X
        obj.y = SLINGSHOT_Y

    def pull(self, mouse_pos):
        if self.object:
            # Calculate distance from anchor
            dx = SLINGSHOT_X - mouse_pos[0]
            dy = SLINGSHOT_Y - mouse_pos[1]
            distance = math.hypot(dx, dy)
            
            # Limit the elastic length
            if distance >= MAX_ELASTIC_LENGTH:
                scale = MAX_ELASTIC_LENGTH / distance
                self.loaded_pos = (SLINGSHOT_X - dx * scale, SLINGSHOT_Y - dy * scale)
            else:
                self.loaded_pos = mouse_pos
            
            # Update object position
            self.object.x = self.loaded_pos[0]
            self.object.y = self.loaded_pos[1]
            
            # Calculate trajectory preview
            self.calculate_trajectory()

    def release(self):
        if self.object and self.loaded_pos:
            # Calculate the pull vector (from pulled position to slingshot center)
            pull_dx = SLINGSHOT_X - self.loaded_pos[0]
            pull_dy = SLINGSHOT_Y - self.loaded_pos[1]
            stretch_length = math.hypot(pull_dx, pull_dy)
            
            if stretch_length == 0:
                return
            
            # Calculate velocity proportional to stretch
            velocity = min(stretch_length * 6, 800)  # Cap maximum velocity
            
            # Launch angle - same calculation as trajectory preview
            # Negative pull_dy because screen y coordinates are inverted
            angle = math.degrees(math.atan2(-pull_dy, pull_dx))
            
            self.object.set_velocity(velocity, angle)
            
            self.object = None
            self.loaded_pos = None
            self.trajectory_points = []  # Clear trajectory after release

def check_ball_ball_collision(ball1, ball2):
    if not (ball1.visible and ball2.visible):
        return False
        
    dx = ball2.x - ball1.x
    dy = ball2.y - ball1.y
    distance = math.sqrt(dx ** 2 + dy ** 2)

    if distance < ball1.radius + ball2.radius:
        # Find collision angle
        angle = math.atan2(dy, dx)

        # Velocity components along the collision axis
        v1 = ball1.vx * math.cos(angle) + ball1.vy * math.sin(angle)
        v2 = ball2.vx * math.cos(angle) + ball2.vy * math.sin(angle)

        # Post-collision velocities
        v1_final = ((ball1.mass - ELASTICITY * ball2.mass) * v1 +
                    (1 + ELASTICITY) * ball2.mass * v2) / (ball1.mass + ball2.mass)
        v2_final = ((ball2.mass - ELASTICITY * ball1.mass) * v2 +
                    (1 + ELASTICITY) * ball1.mass * v1) / (ball1.mass + ball2.mass)

        # Update velocities
        ball1.vx = v1_final * math.cos(angle)
        ball1.vy = v1_final * math.sin(angle)
        ball2.vx = v2_final * math.cos(angle)
        ball2.vy = v2_final * math.sin(angle)

        # Resolve overlap
        overlap = 0.5 * (ball1.radius + ball2.radius - distance)
        ball1.x -= overlap * math.cos(angle)
        ball1.y -= overlap * math.sin(angle)
        ball2.x += overlap * math.cos(angle)
        ball2.y += overlap * math.sin(angle)
        return True
    return False

def check_ball_block_collision(ball, block):
    if not (ball.visible and block.visible):
        return False

    # AABB check for broad-phase collision detection
    if (ball.x + ball.radius < block.x or
        ball.x - ball.radius > block.x + block.width or
        ball.y + ball.radius < block.y or
        ball.y - ball.radius > block.y + block.height):
        return False
        
    # Find closest point on block to ball
    closest_x = max(block.x, min(ball.x, block.x + block.width))
    closest_y = max(block.y, min(ball.y, block.y + block.height))
    
    dx = ball.x - closest_x
    dy = ball.y - closest_y
    distance = math.sqrt(dx ** 2 + dy ** 2)

    if distance < ball.radius:
        # Calculate impact speed for damage
        impact_speed = math.hypot(ball.vx, ball.vy)
        damage = (impact_speed * ball.mass) / 1000  # Adjusted for more reasonable damage
        block.take_damage(damage)

        # Point of impact
        point_of_impact = (closest_x, closest_y)
        
        # Impulse is the change in momentum
        impulse_x = -ball.vx * ball.mass * (1 + ELASTICITY)
        impulse_y = -ball.vy * ball.mass * (1 + ELASTICITY)

        block.apply_impulse(impulse_x, impulse_y, point_of_impact)
        
        # Ball bounces off
        ball.vx *= -ELASTICITY
        ball.vy *= -ELASTICITY

        # Resolve overlap
        if distance > 0:
            overlap = ball.radius - distance
            ball.x += overlap * (dx / distance)
            ball.y += overlap * (dy / distance)
        else: # If ball is inside the block
            ball.x += ball.radius
            
        return True
    return False

def check_block_block_collision(block1, block2):
    if not (block1.visible and block2.visible):
        return False
        
    # Simplified AABB collision (not accounting for rotation in collision detection)
    if (block1.x + block1.width > block2.x and
        block1.x < block2.x + block2.width and
        block1.y + block1.height > block2.y and
        block1.y < block2.y + block2.height):
        
        # Calculate collision point (approximate)
        collision_x = (max(block1.x, block2.x) + min(block1.x + block1.width, block2.x + block2.width)) / 2
        collision_y = (max(block1.y, block2.y) + min(block1.y + block1.height, block2.y + block2.height)) / 2
        
        # Calculate relative velocity
        relative_vx = block1.vx - block2.vx
        relative_vy = block1.vy - block2.vy
        collision_speed = math.hypot(relative_vx, relative_vy)
        
        # Apply damage based on collision force
        if collision_speed > 30:
            damage_factor = 0.05
            damage1 = (collision_speed * block2.mass) / (block1.mass + block2.mass) * damage_factor
            damage2 = (collision_speed * block1.mass) / (block1.mass + block2.mass) * damage_factor
            block1.take_damage(damage1)
            block2.take_damage(damage2)
        
        # Simple collision response
        total_mass = block1.mass + block2.mass
        
        # New velocities
        v1x_new = ((block1.mass - block2.mass) * block1.vx + 2 * block2.mass * block2.vx) / total_mass
        v1y_new = ((block1.mass - block2.mass) * block1.vy + 2 * block2.mass * block2.vy) / total_mass
        v2x_new = ((block2.mass - block1.mass) * block2.vx + 2 * block1.mass * block1.vx) / total_mass
        v2y_new = ((block2.mass - block1.mass) * block2.vy + 2 * block1.mass * block1.vy) / total_mass
        
        block1.vx, block1.vy = v1x_new, v1y_new
        block2.vx, block2.vy = v2x_new, v2y_new
        
        # Find overlap and separate blocks
        overlap_x = min(block1.x + block1.width - block2.x, block2.x + block2.width - block1.x)
        overlap_y = min(block1.y + block1.height - block2.y, block2.y + block2.height - block1.y)
        
        # Push blocks apart
        if overlap_x < overlap_y:
            push = overlap_x / 2
            if block1.x < block2.x:
                block1.x -= push
                block2.x += push
            else:
                block1.x += push
                block2.x -= push
        else:
            push = overlap_y / 2
            if block1.y < block2.y:
                block1.y -= push
                block2.y += push
            else:
                block1.y += push
                block2.y -= push
        
        return True
    return False

def check_all_collisions(balls, blocks, score):
    # Ball-to-Ball collisions
    for i in range(len(balls)):
        for j in range(i + 1, len(balls)):
            if check_ball_ball_collision(balls[i], balls[j]):
                score += 10

    # Ball-to-Block collisions
    for ball in balls:
        for block in blocks:
            if check_ball_block_collision(ball, block):
                score += 5
                if not block.visible:  # Block destroyed
                    score += 50

    # Block-to-Block collisions
    for i in range(len(blocks)):
        for j in range(i + 1, len(blocks)):
            check_block_block_collision(blocks[i], blocks[j])
    
    return score

def draw_background(screen):
    # Draw gradient sky
    for i in range(100):
        r = min(120 + i * 2, 207)
        g = min(221 + i * 0.13, 234)
        b = min(250 - i * 0.03, 247)
        pygame.draw.rect(screen, (r, g, b), (0, i * 7, WIN_WIDTH, 7))
    
    # Draw ground
    pygame.draw.rect(screen, GROUND_COLOR, (0, WIN_HEIGHT - GROUND_HEIGHT, WIN_WIDTH, 30))
    
    # Draw grass
    pygame.draw.rect(screen, GRASS_COLOR, (0, WIN_HEIGHT - GROUND_HEIGHT, WIN_WIDTH, 10))

def main():
    screen = pygame.display.set_mode((WIN_WIDTH, WIN_HEIGHT))
    pygame.display.set_caption("Angry Birds")
    clock = pygame.time.Clock()
    font = pygame.font.Font(None, 36)
    big_font = pygame.font.Font(None, 72)
    
    # Initialize game objects with different health values
    blocks = [
        Block(620, WIN_HEIGHT - GROUND_HEIGHT - 100, 20, 100, 400, BROWN, max_health=150),
        Block(880, WIN_HEIGHT - GROUND_HEIGHT - 100, 20, 100, 400, BLACK, max_health=150),
        Block(750, WIN_HEIGHT - GROUND_HEIGHT - 100, 20, 100, 400, BLACK, max_health=150),
        Block(815, WIN_HEIGHT - GROUND_HEIGHT - 100, 20, 100, 400, BLACK, max_health=150),
        Block(820, WIN_HEIGHT - GROUND_HEIGHT - 250, 20, 100, 100, RED, max_health=80),
        Block(750, WIN_HEIGHT - GROUND_HEIGHT - 250, 20, 100, 100, GREEN, max_health=80),
        Block(750, WIN_HEIGHT - GROUND_HEIGHT - 300, 260, 30, 100, WHITE, max_health=120),
        Block(700, WIN_HEIGHT - GROUND_HEIGHT - 350, 20, 100, 50, BLUE, max_health=60),
        Block(780, WIN_HEIGHT - GROUND_HEIGHT - 350, 20, 100, 50, YELLOW, max_health=60),
        Block(745, WIN_HEIGHT - GROUND_HEIGHT - 400, 150, 30, 5, LIGHTBLUE, max_health=40),
    ]
    
    balls = [
        Ball(SLINGSHOT_X, SLINGSHOT_Y, 10, 300, RED, loaded=True),
        Ball(700, WIN_HEIGHT - 300, 15, 10, GREEN),
        Ball(800, WIN_HEIGHT - 100, 15, 10, GREEN),
    ]
    
    slingshot = Slingshot()
    current_ball = balls[0]
    slingshot.attach_object(current_ball)
    
    score = 0
    bird_count = 0
    dragging = False
    running = True
    game_over = False
    
    while running:
        dt = clock.tick(FPS) / 1000.0 if FPS > 0 else 0
        
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                running = False
            elif event.type == pygame.MOUSEBUTTONDOWN and not game_over:
                if not dragging and slingshot.object:
                    dragging = True
            elif event.type == pygame.MOUSEBUTTONUP and not game_over:
                if dragging and slingshot.object:
                    slingshot.release()
                    dragging = False
            elif event.type == pygame.KEYDOWN:
                if event.key == pygame.K_SPACE and game_over:
                    running = False
        
        if dragging and not game_over:
            mouse_pos = pygame.mouse.get_pos()
            slingshot.pull(mouse_pos)
        
        if not game_over:
            # Update physics
            for ball in balls:
                ball.move()
            for block in blocks:
                block.move()
            
            # Check collisions
            score = check_all_collisions(balls, blocks, score)
            
            # Check if current ball is dead and spawn new one
            if current_ball and current_ball.is_dead() and not slingshot.object:
                bird_count += 1
                if bird_count < 5:
                    new_ball = Ball(SLINGSHOT_X, SLINGSHOT_Y, 10, 100, RED, loaded=True)
                    balls.append(new_ball)
                    slingshot.attach_object(new_ball)
                    current_ball = new_ball
                else:
                    game_over = True
        
        # Draw everything
        draw_background(screen)
        slingshot.draw(screen)
        
        for block in blocks:
            block.draw(screen)
        for ball in balls:
            ball.draw(screen)
        
        # Draw UI
        score_text = font.render(f"Score: {score}", True, BLACK)
        screen.blit(score_text, (50, 50))
        
        birds_left_text = font.render(f"Birds Left: {max(0, 5 - bird_count)}", True, BLACK)
        screen.blit(birds_left_text, (50, 90))
        
        # Draw game over message
        if game_over:
            overlay = pygame.Surface((WIN_WIDTH, WIN_HEIGHT), pygame.SRCALPHA)
            overlay.fill((0, 0, 0, 128))
            screen.blit(overlay, (0, 0))
            
            if score > 1000:
                message = "Yeppp, You Won!"
                color = GREEN
            else:
                message = "Try Again Next Time"
                color = RED
            
            text = big_font.render(message, True, color)
            text_rect = text.get_rect(center=(WIN_WIDTH // 2, WIN_HEIGHT // 2))
            screen.blit(text, text_rect)
            
            continue_text = font.render("Press SPACE to exit", True, WHITE)
            continue_rect = continue_text.get_rect(center=(WIN_WIDTH // 2, WIN_HEIGHT // 2 + 60))
            screen.blit(continue_text, continue_rect)
        
        pygame.display.flip()
    
    pygame.quit()

if __name__ == "__main__":
    main()
