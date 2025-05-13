import pygame
import random
import math
import colorsys

pygame.init()

screen_width = 500
screen_height = 500
screen = pygame.display.set_mode((screen_width, screen_height))
pygame.display.set_caption("Бессмысленная анимация")

WHITE = (255, 255, 255)
BLACK = (0, 0, 0)
GRAY = (100, 100, 100)

clock = pygame.time.Clock()

GRAVITY_BALL = pygame.Vector2(0, 0.3)
FRICTION_BALL = 0.99
GRAVITY_PARTICLE = pygame.Vector2(0, 0.08)
FRICTION_PARTICLE = 0.98
AIR_RESISTANCE_PARTICLE = 0.995
PARTICLE_PUSH_FACTOR = 0.08
TRAIL_LENGTH = 12
STAR_COUNT = 50
FIXED_CIRCLE_THICKNESS = 8

enable_mouse_force = False

def get_bright_color(hue_shift=0):
    hue = (random.random() + hue_shift) % 1.0
    saturation = random.uniform(0.8, 1.0)
    value = random.uniform(0.9, 1.0)
    r, g, b = colorsys.hsv_to_rgb(hue, saturation, value)
    return (int(r * 255), int(g * 255), int(b * 255))

class Particle:
    def __init__(self, pos, vel, color, lifespan, size=2):
        self.pos = pygame.Vector2(pos)
        self.vel = pygame.Vector2(vel)
        self.color = color
        self.initial_lifespan = max(1, lifespan)
        self.current_lifespan = lifespan
        self.size = size
        self.alpha = 255

    def update(self):
        self.vel *= AIR_RESISTANCE_PARTICLE
        self.vel += GRAVITY_PARTICLE
        self.vel *= FRICTION_PARTICLE
        self.pos += self.vel
        self.current_lifespan -= 1

        if self.current_lifespan > 0:
            life_ratio = self.current_lifespan / self.initial_lifespan
            self.alpha = max(0, int(255 * life_ratio))
            if random.random() < 0.1:
                 self.alpha = max(0, self.alpha - random.randint(10, 30))
        else:
            self.alpha = 0

        return self.current_lifespan > 0

    def draw(self, surface):
        if self.alpha > 0 and self.size >= 1:
            try:
                particle_surf = pygame.Surface((int(self.size * 2), int(self.size * 2)), pygame.SRCALPHA)
                draw_color = (*self.color, self.alpha)
                pygame.draw.circle(particle_surf, draw_color, (int(self.size), int(self.size)), int(self.size))
                surface.blit(particle_surf, (int(self.pos.x - self.size), int(self.pos.y - self.size)))
            except ValueError:
                pass


class Circle:
    def __init__(self, center, radius, thickness, color):
        self.center = pygame.Vector2(center)
        self.radius = radius
        self.thickness = thickness
        self.color = color
        self.shrink_speed = 0.6

    def draw(self, surface):
        if self.radius > self.thickness / 2.0:
            draw_thickness = min(max(1, int(self.thickness)), int(self.radius))
            if draw_thickness * 2 > self.radius :
                 draw_thickness = int(self.radius) if self.radius >=1 else 0

            if draw_thickness > 0:
                 try:
                     pygame.draw.circle(
                        surface,
                        self.color,
                        (int(self.center.x), int(self.center.y)),
                        int(self.radius),
                        draw_thickness
                    )
                 except ValueError:
                     pass

    def update(self):
        self.radius -= self.shrink_speed
        if self.radius <= self.thickness / 2.0:
             return 'destroy'
        return True

def check_collision(ball_pos, ball_radius, circle_center, circle_radius, circle_thickness):
    if circle_radius <= circle_thickness / 2.0:
        return False
    distance_vec = ball_pos - circle_center
    distance = distance_vec.length()
    if distance < 1e-6: return False

    outer_radius = circle_radius
    inner_radius = max(0, circle_radius - circle_thickness)

    collides_outer = distance <= outer_radius + ball_radius
    collides_inner = distance >= inner_radius - ball_radius

    return collides_outer and collides_inner and inner_radius < outer_radius

def spawn_particles(pos, base_vel, color, num_particles, min_speed, max_speed, min_life, max_life, min_size, max_size, particle_list, outward_push_vel=None):
    ball_current_pos = ball_pos
    for _ in range(num_particles):
        angle = random.uniform(0, 2 * math.pi)
        speed = random.uniform(min_speed, max_speed)
        p_vel = pygame.Vector2(math.cos(angle), math.sin(angle)) * speed + base_vel
        if outward_push_vel and outward_push_vel.length_squared() > 0:
             # Push away from ball center
            push_dir = (pos - ball_current_pos).normalize() if (pos - ball_current_pos).length_squared() > 1e-6 else pygame.Vector2(random.uniform(-1,1), random.uniform(-1,1)).normalize()
            p_vel += push_dir * outward_push_vel.length() * PARTICLE_PUSH_FACTOR

        lifespan = random.randint(min_life, max_life)
        size = random.uniform(min_size, max_size)
        particle_list.append(Particle(pos, p_vel, color, lifespan, size))


def spawn_particles_from_circle(circle, particle_list, ball_vel_on_impact):
    num_particles = random.randint(25, 40)
    spawn_radius_outer = circle.radius
    spawn_radius_inner = max(0, circle.radius - circle.thickness)
    ball_current_pos = ball_pos

    if spawn_radius_inner >= spawn_radius_outer:
        return

    for i in range(num_particles):
        angle = random.uniform(0, 2 * math.pi)
        spawn_radius = random.uniform(spawn_radius_inner, spawn_radius_outer)
        px = circle.center.x + spawn_radius * math.cos(angle)
        py = circle.center.y + spawn_radius * math.sin(angle)
        p_pos = pygame.Vector2(px, py)

        outward_angle = math.atan2(py - circle.center.y, px - circle.center.x)
        base_speed = random.uniform(1.2, 2.8)
        p_vel = pygame.Vector2(math.cos(outward_angle), math.sin(outward_angle)) * base_speed


        push_dir = (p_pos - ball_current_pos).normalize() if (p_pos - ball_current_pos).length_squared() > 1e-6 else pygame.Vector2(math.cos(outward_angle), math.sin(outward_angle))
        p_vel += push_dir * ball_vel_on_impact.length() * PARTICLE_PUSH_FACTOR

        p_vel += pygame.Vector2(random.uniform(-0.4, 0.4), random.uniform(-0.4, 0.4))

        lifespan = random.randint(40, 90)
        size = random.uniform(1.5, 3.5)
        particle_list.append(Particle(p_pos, p_vel, circle.color, lifespan, size))


ball_pos = pygame.Vector2(random.randint(100, screen_width - 100), random.randint(100, screen_height - 100))
original_ball_radius = 10
current_ball_radius = original_ball_radius
ball_vel = pygame.Vector2(random.choice([-3, 3]), random.choice([-3, 3]))
ball_trail = []

is_animating_ball = False
animation_phase = None
animation_max_radius = original_ball_radius * 1.7
expand_speed = 1.1
contract_speed = 0.6

circles = []
particles = []
stars = [(random.randint(0, screen_width), random.randint(0, screen_height), random.uniform(0.5, 1.5)) for _ in range(STAR_COUNT)]


circles.append(
    Circle(
        center=(screen_width // 2, screen_height // 2),
        radius=screen_width // 2 - 10,
        thickness=FIXED_CIRCLE_THICKNESS,
        color=get_bright_color()
    )
)

timer = 0
time_to_new_circle = 27

running = True
while running:

    dt = clock.tick(60) / 1000.0

    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            running = False
        if event.type == pygame.KEYDOWN:
             if event.key == pygame.K_f:
                 enable_mouse_force = not enable_mouse_force
        if event.type == pygame.MOUSEBUTTONDOWN:
            mouse_pos = pygame.Vector2(event.pos)
            if enable_mouse_force:
                force_dir = mouse_pos - ball_pos
                if force_dir.length_squared() > 1e-6:
                     force = force_dir.normalize() * 15.0
                     ball_vel += force
            else:
                 burst_color = get_bright_color(hue_shift=0.5)
                 spawn_particles(mouse_pos, pygame.Vector2(0,0), burst_color, 50, 1.0, 3.5, 30, 60, 1, 3, particles, pygame.Vector2(0,0))


    timer += 1
    if timer >= time_to_new_circle:
        circles.append(
            Circle(
                center=(screen_width // 2, screen_height // 2),
                radius=screen_width // 2 - 10,
                thickness=FIXED_CIRCLE_THICKNESS,
                color=get_bright_color()
            )
        )
        timer = 0



    ball_vel += GRAVITY_BALL
    ball_vel *= (FRICTION_BALL)
    ball_pos += ball_vel


    bounce_factor = 0.85
    if ball_pos.x - current_ball_radius < 0:
        ball_pos.x = current_ball_radius
        ball_vel.x *= -bounce_factor
    elif ball_pos.x + current_ball_radius > screen_width:
        ball_pos.x = screen_width - current_ball_radius
        ball_vel.x *= -bounce_factor
    if ball_pos.y - current_ball_radius < 0:
        ball_pos.y = current_ball_radius
        ball_vel.y *= -bounce_factor
    elif ball_pos.y + current_ball_radius > screen_height:
        ball_pos.y = screen_height - current_ball_radius
        ball_vel.y *= -bounce_factor
        if abs(ball_vel.y) < 1.5:
             ball_vel.y = -1.5 * bounce_factor


    collided_this_frame = False
    circles_to_remove = set()

    for c in circles:
        status = c.update()

        if status == 'destroy':
            if c not in circles_to_remove:
                 spawn_particles_from_circle(c, particles, pygame.Vector2(0,0))
                 circles_to_remove.add(c)
            continue

        if not collided_this_frame and c not in circles_to_remove:
            if check_collision(ball_pos, current_ball_radius, c.center, c.radius, c.thickness):
                collided_this_frame = True
                collision_vel_snapshot = ball_vel.copy()
                distance_vec = ball_pos - c.center
                distance = distance_vec.length()

                if distance > 1e-6:
                    normal = distance_vec.normalize()

                    outer_radius = c.radius
                    inner_radius = max(0, c.radius - c.thickness)
                    penetration = 0
                    reflect_normal = normal

                    dist_to_outer = abs(distance - outer_radius)
                    dist_to_inner = abs(distance - inner_radius)

                    if dist_to_outer < dist_to_inner :
                        penetration = (outer_radius + current_ball_radius) - distance
                        if penetration > 0:
                            ball_pos += normal * penetration
                        reflect_normal = normal
                    else:
                        penetration = distance - (inner_radius - current_ball_radius)
                        if penetration > 0:
                            ball_pos -= normal * penetration
                        reflect_normal = -normal

                    if ball_vel.dot(reflect_normal) < 0:
                        ball_vel = ball_vel.reflect(reflect_normal)
                        impulse = 1.24
                        ball_vel *= impulse


                    if not is_animating_ball:
                        is_animating_ball = True
                        animation_phase = 'expanding'

                    if c not in circles_to_remove:
                        spawn_particles_from_circle(c, particles, collision_vel_snapshot)
                        circles_to_remove.add(c)

    circles = [c for c in circles if c not in circles_to_remove]

    if is_animating_ball:
        if animation_phase == 'expanding':
            current_ball_radius += expand_speed
            if current_ball_radius >= animation_max_radius:
                current_ball_radius = animation_max_radius
                animation_phase = 'contracting'
        elif animation_phase == 'contracting':
            current_ball_radius -= contract_speed
            if current_ball_radius <= original_ball_radius:
                current_ball_radius = original_ball_radius
                is_animating_ball = False
                animation_phase = None

    ball_trail.append(ball_pos.copy())
    if len(ball_trail) > TRAIL_LENGTH:
        ball_trail.pop(0)

    particles = [p for p in particles if p.update()]


    screen.fill(BLACK)

    for x, y, size in stars:
        alpha = random.randint(50, 150)
        pygame.draw.circle(screen, (*GRAY, alpha) , (x, y), size)


    for i, pos in enumerate(ball_trail):
        alpha = int(150 * (i / TRAIL_LENGTH))
        radius = int(current_ball_radius * (i / TRAIL_LENGTH) * 0.5)
        if radius >= 1:
            trail_surf = pygame.Surface((radius * 2, radius * 2), pygame.SRCALPHA)
            pygame.draw.circle(trail_surf, (WHITE[0], WHITE[1], WHITE[2], alpha), (radius, radius), radius)
            screen.blit(trail_surf, (int(pos.x - radius), int(pos.y - radius)))


    for c in circles:
        c.draw(screen)


    for p in particles:
        p.draw(screen)


    pygame.draw.circle(screen, WHITE, (int(ball_pos.x), int(ball_pos.y)), int(current_ball_radius))

    if enable_mouse_force:
         font = pygame.font.SysFont(None, 24)
         text_surf = font.render('Mouse Force: ON (Press F to toggle)', True, WHITE)
         screen.blit(text_surf, (10, 10))


    pygame.display.flip()

pygame.quit()
