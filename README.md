# libraries - using p5 and math for drawing
	from p5 import *
	import random, math

# window sizes
	win = {'width': 400, 'height': 600}
	FLOOR_Y = win['height'] - 40
	TWO_PI = math.pi * 2 #math constant
	
# foods hierarchy from small -> big - all these are interchangeble to make your own personalized foods
	foods_order = ['mint','pea', 'tomato', 'egg', 'pumpkin', 'cookie', 'sushi', 'pie', 'mooncake', 'pizza']
	food_colors = {
	  'mint': Color(180, 255, 220),
	  'pea': Color(100, 220, 100), 
	  'tomato': Color(220, 60, 60),
	  'egg': Color(255, 255, 200),
	  'pumpkin': Color(255, 180, 60),
	  'cookie': Color(200, 170, 110),
	  'sushi': Color(240, 230, 200),
	  'pie': Color(240, 200, 110),
	  'mooncake': Color(235, 190, 70),
	  'pizza': Color(250, 170, 60)
	}
	food_size = {'mint':20,'pea':30,'tomato':40, 'egg':70, 'pumpkin':90, 'cookie':120,'sushi': 170, 'pie': 190,'mooncake':200,'pizza':210}

# physics

	GRAVITY = 0.32
	REST    = 0.55      #lower= much less bounce
	AIR_DAMP= 0.995     #gentle global damping
	FLOOR_FRICTION = 0.98
	ITER    = 3         #collision iterations

	#merge on touch just like in suika
	MERGE_TOL   = 3.0   #extra pixels of distance so that its merging more sensitive


# game state - appendeble
	
	foods = []           #all falling foods
	current_food = None  #the one movingo n the x axis
	score = 0
	you_win = False #make later
	game_over = False #make later


# helpers

	def clamp(x, lo, hi): #clamp is used for min max values
	  if x < lo: return lo #force within bounds
	  if x > hi: return hi 
	  return x
	
	def next_tier(name):
	  i = foods_order.index(name) #find index of current food
	  return foods_order[min(i+1, len(foods_order)-1)] #return next food or same if at top
	
	def make_food(name, x, y): #create a food dict
	  r = food_size[name] / 2 #radius!
	  m = (r*r)/80.0 + 1.0   # rough mass/area
	  return {
	    'name': name, 
	    'x': float(x), 'y': float(y), #this is position for food 
	    'vx': 0.0, 'vy': 0.0, #velocity
	    'd': r*2, 'r': r, 'm': m, #diameter, radius, mass
	    'falling': True, 
	    'cool': 0,    #frames before it can merge so 0 means ready
	  }


# setup - for p5 required
	def setup(): 
	  size(win['width'], win['height'])
	  title("mirandas dysfunctional suikia! click to drop fruit, merges on touch") 
	  spawn_new_food() #call the food function to spawn food at start

# new Food
	def spawn_new_food():
	  global current_food 
	  name = foods_order[random.randint(0, 3)]   # only spawn small foods
	  current_food = make_food(name, random.randint(80, win['width'] - 80), 60) #spawn near top
	  current_food['falling'] = False #not until mouse pressed
	  current_food['vx'] = random.uniform(-5, 5) #move on the x axis
	  current_food['vy'] = 0.0 #this is y velocity which is not moving yet
	  current_food['cool'] = 0 #ready to merge right away

# mouse pressed - drops food when mouse pressed

	def mouse_pressed():
	  global current_food, foods
	  if current_food and not current_food['falling']:
	    dropping = current_food 
	    dropping['falling'] = True #update game state
	    dropping['vy'] = 2.0
	    dropping['cool'] = 0   #still ready to merge right away
	    foods.append(dropping) #add to foods list
	    current_food = None
	    spawn_new_food()       #food spawns immediately after drop

# lose, does not work yet
	def lose(): #make this later 
	  global game_over
	  game_over = True
	  print("you lose final score:", score)


  # damping - inspo from Byron
	def physics_step(f):
	  f['vx'] *= AIR_DAMP #air resistance makes it slow down
	  f['vy'] *= AIR_DAMP #same for y axis

  # gravity +integrate
	  f['vy'] += GRAVITY #gravity effect so it falls down with increasing speed
	  f['x'] += f['vx']; f['y'] += f['vy'] #update position

  # walls
	  if f['x'] - f['r'] < 0: #left wall for bounce
	    f['x'] = f['r']; f['vx'] *= -REST #bouncing effect to reverse velocity
	  if f['x'] + f['r'] > win['width']: #right wall for bounce
	    f['x'] = win['width'] - f['r']; f['vx'] *= -REST #this is to reverse velocity on compared to rest, it loses some energy on bounce

  # floor
	  if f['y'] + f['r'] > FLOOR_Y: #floor collision
	    f['y'] = FLOOR_Y - f['r'] #reset position to be on floor
	    f['vy'] *= -REST #reverse y velocity with bounce effect
	    f['vx'] *= FLOOR_FRICTION #friction effect on x velocity
	    if abs(f['vy']) < 0.4: #if abs means absolute value, small bounce threshold
	      f['vy'] = 0.0 #stop bouncing 

  # cooldown tick
	  if f['cool'] > 0: #cooldown for merging
	    f['cool'] -= 1 

	def resolve_pair(a, b): #this is to make them bounce off each other
  # overlap
  dx = b['x'] - a['x']; dy = b['y'] - a['y'] #this is the distance vector between two foods so we can calculate the distance
  dist = math.hypot(dx, dy) #distance between centers
  min_d = a['r'] + b['r'] #minimum distance to avoid overlap
  if dist == 0: 
    dx, dy = 0.001, 0.0 #this is to avoid division by zero whcih can cause errors
    dist = 0.001
  if dist >= min_d: #no collision so they are separated
    return #return to caller

  nx, ny = dx/dist, dy/dist #a vector is normalized because we want direction only. a vector is a direction with magnitude
  overlap = min_d - dist #this is how much they are overlapping

  # positional correction which is split by mass 
  total = a['m'] + b['m'] #total mass
  a_share = b['m'] / total #portion of movement for a
  b_share = a['m'] / total


  a['x'] -= nx * overlap * a_share #move a out of collision
  a['y'] -= ny * overlap * a_share 
  b['x'] += nx * overlap * b_share
  b['y'] += ny * overlap * b_share #quick fix for sinking

  # relative velocity along normal
  rvx = b['vx'] - a['vx']; rvy = b['vy'] - a['vy'] #relative velocity
  rel_n = rvx*nx + rvy*ny #relative velocity in normal direction
  if rel_n > 0: #they are moving apart
    return  #separates

  # normal impulse for tempered bounce
  j = -(1 + REST) * rel_n 
  j /= (1.0/a['m'] + 1.0/b['m']) #this is for mass effect
  impx, impy = j*nx, j*ny
  a['vx'] -= impx / a['m']; a['vy'] -= impy / a['m']
  b['vx'] += impx / b['m']; b['vy'] += impy / b['m'] #apply impulse

  
  # resolve all collisions between foods
def resolve_all_collisions(): 
  n = len(foods) #number of foods
  for i in range(n): #iterate through all foods
    for j in range(i+1, n): 
      resolve_pair(foods[i], foods[j])


# merging on touch

def can_merge(a, b):
  if a['name'] != b['name']: #must be same type to merge
    return False
  #overlap with a little tolerance
  dx = b['x'] - a['x']; dy = b['y'] - a['y']
  return math.hypot(dx, dy) <= (a['r'] + b['r'] + MERGE_TOL) and a['cool'] == 0 and b['cool'] == 0 #ready to merge

# merging mechanisms
def merge_pass():
  global foods, you_win, score #you_win condition has not been implemented yet -> make later
  changed = False #tracking if any merges happens
  n = len(foods) #number of foods
  to_add = [] #new foods to add
  gone = set() #this is to track merged foods


# continous merging in one pass, also win condition and score keeping
  for i in range(n):
    if i in gone: continue #already merged
    for j in range(i+1, n):
      if j in gone: continue 
      a, b = foods[i], foods[j] #foods to check
      if can_merge(a, b):
        new_name = next_tier(a['name'])
        if new_name == 'pizza':
          you_win = True
        mx = (a['x'] + b['x']) / 2.0
        my = (a['y'] + b['y']) / 2.0
        nf = make_food(new_name, mx, my)
  # momentum with small pop with merging
        nf['vx'] = (a['vx'] + b['vx']) / 2.0 #momentum conservation
        nf['vy'] = (a['vy'] + b['vy']) / 2.0 - 0.4 #small pop up
        nf['cool'] = 6   # prevent instant re-merge
        nf['falling'] = True 
        to_add.append(nf)
        gone.add(i); gone.add(j)
        score += food_size[new_name]
        changed = True
        break

  if changed: 
    foods = [f for k,f in enumerate(foods) if k not in gone] + to_add #rebuild food list
  return changed


# food designs
def draw_food_design_centered(f):
  r = f['r']; name = f['name'] #food name determines design
  if name == 'pea':
    no_fill(); stroke(60,160,60)  #just a circle for pea


  elif name == 'tomato':
    fill(60,150,60)
    for i in range(2):
      ang = i*(TWO_PI/5)+0.3 #angle is used to position the leaves
      triangle((0,0),
               (r*0.65*math.cos(ang-0.2), r*0.65*math.sin(ang-0.2)), #this is the tomato leaves, using math cos and sin to position and rotate
               (r*0.65*math.cos(ang+0.2), r*0.65*math.sin(ang+0.2))) #other side
    fill(255,255,255,90)

  elif name == 'mint':
    fill(180, 255, 220)
 
  elif name == 'egg':
    fill(255, 255, 200)
    ellipse((0,0), r*1.6, r*1.2) #egg white
    fill(255, 255, 100)
    ellipse((0,0), r*0.9, r*0.7) #egg yolk


  elif name == 'cookie':
    fill(90,60,40)
    for (ox,oy) in [(-0.3,-0.2),(0.2,-0.15),(-0.1,0.1),(0.28,0.18),(-0.25,0.25)]: #chocolate chips
      ellipse((ox*r, oy*r), r*0.18, r*0.18) #this is to draw the chocolate chips on the cookie

  elif name == 'sushi':
    no_stroke(); fill(255,179,138) #salmon color
    ellipse((0,0) , r*1.6, r*1.2)
    no_fill(); stroke(1, 50, 32); stroke_weight(6) #seaweed color
    ellipse((0,0), r*1.6 - 4, r*1.2 - 4)
    stroke_weight(1); no_stroke(); fill(144, 238, 144) #rice color
    rect((-r*0.3, -r*0.2), r*0.4, r*0.4) #avacado piece


  elif name == 'pie':
    no_fill(); stroke(200,150,80); stroke_weight(4) #pie crust
    circle((0,0), (r*2)-4)
    stroke_weight(1); no_stroke(); fill(220,170,90,120) #pie filling
    for k in range(3): #draw pie slices
      rect((k*r*0.22 - r*0.05, -r*0.9), r*0.1, r*1.8) #vertical slice
      rect((-r*0.9, k*r*0.22 - r*0.05), r*1.8, r*0.1) #horizontal slice

  elif name == 'pumpkin':
    no_stroke(); fill(255,180,60) 
    ellipse((0,0), r*1.6, r*1.2)
    fill(200, 150, 50)
    for i in range(3): #pumpkin ridges
      ang = i*(TWO_PI/5)+0.1 #angle for ridges
      triangle((0,0), #center point
               (r*0.65*math.cos(ang-0.2), r*0.65*math.sin(ang-0.2)), #pumpkin ridges
               (r*0.65*math.cos(ang+0.2), r*0.65*math.sin(ang+0.2))) 
    fill(34,139,34)
    rect((-r*0.1, -r*0.8), r*0.2, r*0.15) #pumpkin stem




  elif name == 'mooncake':
    no_stroke(); fill(255,220,120,120)
    circle((0,0), r*1.1)
    no_fill(); stroke(180,130,60)
    for i in range(8): 
      ang = i*(TWO_PI/8) 
      line((r*0.25*math.cos(ang), r*0.25*math.sin(ang)), #lines radiating from center
           (r*0.85*math.cos(ang), r*0.85*math.sin(ang))) #this is the other end of the line
      fill(180,130,60)
   

  elif name == 'pizza':
    no_fill(); stroke(200,140,70); stroke_weight(10)
    circle((0,0), r*2 - 4)
    no_stroke(); stroke_weight(1) 
 
    fill(200,60,60)
    for (ox,oy,s) in [(-0.45,-0.2,0.35),(0.2,-0.25,0.3),(-0.1,0.25,0.33),(0.35,0.22,0.28)]: #pepperoni positions
      ellipse((ox*r, oy*r), r*s, r*s) #draw pepperoni slices
    fill(40,120,40)

# drawing individual food
def draw_food(f): 
  x, y, r = f['x'], f['y'], f['r'] #food position and radius
  #shadow
  no_stroke(); fill(0,0,0,80)
  ellipse((x, y + r*0.2), r*1.6, r*0.5)
#base circle
  fill(food_colors[f['name']]); circle((x,y), r*2) 
  #pop
  push_matrix(); translate(x,y); draw_food_design_centered(f); pop_matrix()
  #label for the foods
  fill(0); text(f['name'], (x - 12, y - 4)) #label position adjustment

def draw_all(): #calling the draw the foods so it can be organized and called in the main draw function
  for f in foods: #loop through all foods
    draw_food(f)
  if current_food: #for the current food to be dropped and previewed
    draw_food(current_food)

def draw_score():
  fill(255); text(f"score: {score}", (10, 10)) #score display


def draw():
  background(30)

  stroke(120); line((0, FLOOR_Y), (win['width'], FLOOR_Y)); no_stroke() #this is the floor line

  # previewing slides across top
  if current_food and not current_food['falling']:
    f = current_food
    f['x'] += f['vx']
    if f['x'] - f['r'] < 0 or f['x'] + f['r'] > win['width']: #bounce off walls
      f['vx'] *= -1.05 #slight speed increase on bounce
      f['vx'] += random.uniform(-0.25, 0.25) #this is to make the movement less predictable (might get rid of later)
    f['x'] = clamp(f['x'], f['r'], win['width'] - f['r']) #clamp within window

  # physics update
  for f in foods:
    physics_step(f) #call physics step for each food

  # merge first (so bouncy contacts don't get lost),then collide,then try merging again
  merged_now = merge_pass()
  for _ in range(ITER): #multiple collision resolutions per frame
    resolve_all_collisions()
  merged_now = merge_pass()

  # call the functions
  draw_all() #call draw all foods
  draw_score() #call score display


run()
