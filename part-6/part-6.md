# Part 6 - Doing Some Damage

In Part 5, we revamped how we make enemies, and laid the framework for attacking. Today we're going to go all the way and start doing more than just kicking them. Get a snack and a drink, this might take a little while. First off, let's change our Entity object to add important information.

```lua
-- entities.lua

...

-- constructor for generic entity
function Entity:new(args)
	local o = setmetatable({}, Entity)
	o.x = args.x
	o.y = args.y
	o.sprite = args.sprite or 0
	o.name = args.name or ""
	o.blocksMovement = args.blocksMovement or false
	o.combatant = args.combatant or false
	o.hp = args.hp or 0
	o.attack = args.attack or 0
	return o
end

-- generic move function for entity
function Entity:move(dx, dy)
	local destX = self.x + dx
	local destY = self.y + dy
	if getBlockingEntity(destX, destY) then
		self:melee(destX, destY)
	elseif isWalkable(destX, destY) then
		self.x = destX
		self.y = destY
	end
end

-- generic melee attack for entity
function Entity:melee(x, y)
	target = getBlockingEntity(x, y)
	if target.combatant and target.hp > 0 then
		target.hp = target.hp - self.attack
		printh(self.name .. " attacks " .. target.name .. " for " .. self.attack .. " damage.")
		if target.hp <= 0 then
			printh(self.name .. " kills " .. target.name .. "!")
			del(entities, target)
		end
	end
end

...

```

So here, we've added a bunch of new properties to our Entity object:
 - `combatant` - This tells us whether this entity should be moving about and fighting.
 - `hp` - This is your standard hit point pool. When it reaches 0, the entity is dead and removed from the entity pool.
 - `attack` - This is how much damage the entity will do on an attack.
You might notice some different stuff about the constructor. First off, we modified the parameters to just accept a table of arguments. This keeps our function looking a little cleaner. We also added a default value to each argument (the `or something` part of the line). That lets us specify how a default Entity should look, so we don't have to always put something for each property.

To deconflict with our new attack property, we've renamed the `attack()` function to `melee()`. Inside the `melee()` function, you'll notice some new logic. If we have a blocking entity, we check to see if it is a combatant and if it has hit points. This way we can implement entities that may be blocking movement, but shouldn't be attacked, like a tree. If our blocking entity is truly a dastardly enemy, we subtract our attack value from its hp. If the hp is at or below zero, we've killed it, and it gets removed from the entity pool. Let's try it out!

![[p6-killing_enemies.gif]]

The GIF leaves something to be desired, but trust me, there were some furious button presses and captivating log messages involved in killing that enemy! As you can see, once an enemy is killed, they get removed from the pool and automatically stop being drawn. You can even walk right into the tile they were willing to give their life to defend.

This is all well and good, but killing defenseless monsters leaves something to be desired. We need to get them moving around and trying to take their revenge! Let's start by taking some pathfinding code from a decent implementation for Pico-8. It's a little out of scope to worry about pathfinding here, so I'll just tell you I'm taking code from [here](https://github.com/morgan3d/misc/tree/master/p8pathfinder) because I felt it looked like a pretty good implementation of A* pathfinding, and I'm putting it in pathing.lua, shown below:

```lua
-- largely taken from the following implementations:
-- https://github.com/morgan3d/misc/tree/master/p8pathfinder
function find_path
(start,
 goal,
 estimate,
 edge_cost,
 neighbors, 
 node_to_id, 
 graph)
 
 -- the final step in the
 -- current shortest path
 local shortest, 
 -- maps each node to the step
 -- on the best known path to
 -- that node
 best_table = {
  last = start,
  cost_from_start = 0,
  cost_to_goal = estimate(start, goal, graph)
 }, {}

 best_table[node_to_id(start, graph)] = shortest

 -- array of frontier paths each
 -- represented by their last
 -- step, used as a priority
 -- queue. elements past
 -- frontier_len are ignored
 local frontier, frontier_len, goal_id, max_number = {shortest}, 1, node_to_id(goal, graph), 32767.99

 -- while there are frontier paths
 while frontier_len > 0 do

  -- find and extract the shortest path
  local cost, index_of_min = max_number
  for i = 1, frontier_len do
   local temp = frontier[i].cost_from_start + frontier[i].cost_to_goal
   if (temp <= cost) index_of_min,cost = i,temp
  end
 
  -- efficiently remove the path 
  -- with min_index from the
  -- frontier path set
  shortest = frontier[index_of_min]
  frontier[index_of_min], shortest.dead = frontier[frontier_len], true
  frontier_len -= 1

  -- last node on the currently
  -- shortest path
  local p = shortest.last
  
  if node_to_id(p, graph) == goal_id then
   -- we're done.  generate the
   -- path to the goal by
   -- retracing steps. reuse
   -- 'p' as the path
   p = {goal}

   while shortest.prev do
    shortest = best_table[node_to_id(shortest.prev, graph)]
    add(p, shortest.last)
   end

   -- we've found the shortest path
   return p
  end -- if

  -- consider each neighbor n of
  -- p which is still in the
  -- frontier queue
  for n in all(neighbors(p, graph)) do
   -- find the current-best
   -- known way to n (or
   -- create it, if there isn't
   -- one)
   local id = node_to_id(n, graph)
   local old_best, new_cost_from_start =
    best_table[id],
    shortest.cost_from_start + edge_cost(p, n, graph)
   
   if not old_best then
    -- create an expensive
    -- dummy path step whose
    -- cost_from_start will
    -- immediately be
    -- overwritten
    old_best = {
     last = n,
     cost_from_start = max_number,
     cost_to_goal = estimate(n, goal, graph)
    }

    -- insert into queue
    frontier_len += 1
    frontier[frontier_len], best_table[id] = old_best, old_best
   end -- if old_best was nil

   -- have we discovered a new
   -- best way to n?
   if not old_best.dead and old_best.cost_from_start > new_cost_from_start then
    -- update the step at this
    -- node
    old_best.cost_from_start, old_best.prev = new_cost_from_start, p
   end -- if
  end -- for each neighbor
  
 end -- while frontier not empty

 -- unreachable, so implicitly
 -- return nil
end

function map_neighbors(node, graph)
 local neighbors = {}
 if isWalkable(node.x, node.y - 1) then 
 	add(neighbors, {x=node.x, y=node.y - 1})
 end
 if isWalkable(node.x, node.y + 1) then 
 	add(neighbors, {x=node.x, y=node.y + 1})
 end
 if isWalkable(node.x - 1, node.y) then 
 	add(neighbors, {x=node.x - 1, y=node.y})
 end
 if isWalkable(node.x + 1, node.y) then 
 	add(neighbors, {x=node.x + 1, y=node.y})
 end
 return neighbors
end

-- estimates the cost from a to
-- b by assuming that the graph
-- is a regular grid and all
-- steps cost 1.
function manhattan_distance(a, b)
 return abs(a.x - b.x) + abs(a.y - b.y)
end

```

That's a lot of code. I'm certainly grateful to the people that make things available to learn from online. We'll add some enemy AI now. Not ChatGPT or anything, just basic 'if i see the player, I'm going to try to kill them' game AI. Head on over to entities.lua.

```lua
-- entities.lua

...

-- ai function to make entities chase the player
function Entity:ai()
	if player and self != player and self.combatant then
		-- give enemies a fov of 7 tiles
		local start = {x=self.x, y=self.y}
		local goal = {x=player.x, y=player.y}
		if insideCircle(start, goal, 7) then
			path = find_path(start, goal,
                  manhattan_distance,
                  function () return 1 end,
                  map_neighbors,
                  function (node) return node.y * 8 + node.x end,
                  nil)
         if path then
         	local p = path[count(path) - 1]
         	-- do a check to see if a blocking entity that isn't the player
         	-- is there, we don't want our enemies killing each other
         	local e = getBlockingEntity(p.x, p.y)
         	if e and e != player then
         		return
         	end
         	local dX = p.x - self.x
         	local dY = p.y - self.y
         	self:move(dX, dY)
         end
		end		
	end
end

...
```

So, in this AI function, we check to see if the player is in existence. Then we check to see if the entity whose brain this is isn't the player (aka an enemy) and it's a combatant. If all that is true, we then check to see if the player is visible to the enemy (just using a circle of 7 tiles for now). We don't want all the enemies in the dungeon swarming us the moment we start. If the player is visible to the enemy, we call off to our pathfinding function to see if there's a way to get to the player. If so, we move to the next closest step in that path, assuming there's no other enemy in that spot. By using the `Entity:move()` function, the melee attack we use gets baked into the enemy movement too, meaning they can totally kill you now. But to get all this to work, we need to give these enemies a chance to use their brains.

```lua
-- main.lua
...
include "pathing.lua"

State = {PLAYER_TURN = "1", ENEMY_TURN = "2"}

...

function _init()
	currentState = State.PLAYER_TURN
	populateMap()
end

function _update()
	if currentState == State.PLAYER_TURN then
		if btnp(0) then
			player:move(-1, 0)
		elseif btnp(1) then
			player:move(1, 0)
		elseif btnp(2) then
			player:move(0, -1)
		elseif btnp(3) then
			player:move(0, 1)
		end
		updateFOV()
		updateMap()
	elseif currentState == State.ENEMY_TURN then
		updateEntities()
		currentState = State.PLAYER_TURN
	end
end

...

function updateEntities()
	for entity in all(entities) do
		entity:ai()
	end
end

```

```lua
-- entities.lua

...

-- generic move function for entity
function Entity:move(dx, dy)
	local destX = self.x + dx
	local destY = self.y + dy
	if getBlockingEntity(destX, destY) then
		self:melee(destX, destY)
	elseif isWalkable(destX, destY) then
		printh(self.name .. " moves x: " .. destX .. ", y: " .. destY)
		self.x = destX
		self.y = destY
	end
	if player then
		currentState = State.ENEMY_TURN
	end
end

...
```
We've added pathing.lua to our imports (important to pull that code into our program), and we've added game states. Right now we only have two, the player turn and the enemy turn. We're not doing any complicated logic with speed or anything, everyone gets one turn at a time. In `_update()`, we check to see whose turn it is. If the player's, it waits for input, and on a move, switches to the enemy turn afterwards. If it's the enemy turn, it iterates through all entities, running the `Entity:ai()` function. After everyone's had a go, it's back to the player. Let's give it a shot!
![[p6-enemy_movement.gif]]

Wow, everything's so lively now! If you wander around fighting enemies long enough, you'll probably even die, which will definitely crash your game. You might notice I bumped up our view radius. That's so we can see enemies just a little bit before they can see us, feel free to tweak to your liking.

Well, what a fulfilling day, and what a good place to take a break. As always, you can give it a try here. I'll see you in Part 7!
