# Part 7 - Building an Interface

Welcome back! In [Part 6](../part-6/part-6.html) we managed to get enemies moving around and finally did some fighting. But a lot of that involved looking at terminal logs using `printh`, and that's not super convenient. It would be great if we could see what was going on inside our game itself. Well, today let's try and figure that out. Let's start out in main.lua.

```lua
-- main.lua

...

function _draw()
	cls()
	clip(0, 0, 380, 200)
	camera(player.x*16 - 11*16, player.y*16 - 6*16)
	map()
	drawEntities()
	drawInterface()
end

function drawInterface()
	clip()
	camera()
	drawSideUI()
	drawBottomUI()
end

function drawSideUI()
	rectfill(382, 2, 478, 198, 1)
	rect(382, 2, 478, 198, 7)
end

function drawBottomUI()
	rectfill(0, 200, 478, 268, 1)
	rect(0, 200, 478, 268, 7)	
end
```
Let's start by making some sections for our information to live. Maybe we'll have a bit on the side for information like player stats, and a bit at the bottom for a message log to see what's going on. To do that, first we change `_draw()` to limit our initial clip area and change our camera position. This limits our map drawing to a 380x200 rectangle, out of our 480x270 screen size. Then after we do all of our usual stuff, we'll call off to a new function `drawInterface()`, which has two helper functions `drawSideUI()` and `drawBottomUI()`. These functions are pretty self explanatory. First, we reset our clipping rectangle and camera, to allow us to reference positions directly on the screen. Then, we draw some blue rectangles with a classy Final Fantasy style white border along the edges.

![[p7-rectangles.png]]
Looking pretty good so far, but a blue rectangle's useless without some words in there. Let's start with the side UI, since the information for that is readily available in our player object.

```lua
-- main.lua

...

function drawSideUI()
	rectfill(382, 2, 478, 198, 1)
	rect(382, 2, 478, 198, 7)
	cursor(386, 6)
	print(player.name)
	print("HP: " .. player.hp)
	print("Attack: " .. player.attack)
end
```
![[p7-player_info.png]]

Well, that certainly does the trick. Now we can see our name, current HP and how much damage we do, with plenty of room for any future stuff we need to plop in there. It's not done in the nicest way, using magic numbers and stuff for our positioning, but there's time for cleaning things up later on. The `cursor()` function is new, it lets us set the position that we start printing text at. after every `print()` function, it jumps down to let us print on the next line for the size of our font. Pretty nice!

For our bottom UI, we're going to need some sort of log handler to let us store all messages and display the most recent ones. We'll put that into a new file called logger.lua.

```lua
-- logger.lua

logs = {}

function log(message)
	add(logs, message)
end

function mostRecentLogs(amount)
	local index = count(logs)
	local recentLogs = {}
	while (amount > 0) do
		if index > 0 then
			add(recentLogs, logs[index])
		end
		amount = amount - 1
		index = index - 1
	end
	return recentLogs
end
```

Real simple, just a table we're dropping strings into, and a function to fetch the most recent few. Back in main.lua, we'll finish up the bottom UI. We'll add a friendly welcome message to the logs on init as well.

```lua
-- main.lua

...

function _init()
	currentState = State.PLAYER_TURN
	populateMap()
	log("Welcome to the dungeon!")
end

...

function drawBottomUI()
logs = {}

function log(message)
	add(logs, message)
end

function mostRecentLogs(amount)
	local index = count(logs)
	local recentLogs = {}
	while (amount > 0) do
		if index > 0 then
			add(recentLogs, logs[index])
		end
		amount = amount - 1
		index = index - 1
	end
	return recentLogs
end
```

And let's pop over to entities.lua for the real meat and potatoes: converting those `printh()` functions to our new `log()` function so that we can see the numbers fly!

```lua
-- entities.lua

...

-- generic melee attack for entity
function Entity:melee(x, y)
	target = getBlockingEntity(x, y)
	if target.combatant and target.hp > 0 then
		target.hp = target.hp - self.attack
		log(self.name .. " attacks " .. target.name .. " for " .. self.attack .. " damage.")
		if target.hp <= 0 then
			log(self.name .. " kills " .. target.name .. "!")
			del(entities, target)
		end
	end
end
```

If you run your game, you should be able to see your logs in all their glory.

![[p7-logs.gif]]

Wow, we're cooking with gas now! What more could you want? Well, maybe some color so we can differentiate the log messages a little? We'll make our player's actions white, enemy actions orange, and deaths red. This is really simple, since Picotron lets us just specify color codes inside the text.

```lua
-- entities.lua

...

-- generic melee attack for entity
function Entity:melee(x, y)
	target = getBlockingEntity(x, y)
	if target.combatant and target.hp > 0 then
		target.hp = target.hp - self.attack
		local c = 7
		if self != player then
			c = 9
		end
		log("\f" .. c .. self.name .. " attacks " .. target.name .. " for " .. self.attack .. " damage.\f7")
		if target.hp <= 0 then
			log("\f8" .. self.name .. " kills " .. target.name .. "!\f7")
			del(entities, target)
		end
	end
end
```

I'm not going to make a new GIF of that, but if you run it, you should see glorious color in your logs. What other things should we add? Well, we probably want a new state for the game for after we're killed. This would let us pop up a window saying 'YOU DEAD' or something. While we're talking about states and windows, maybe something where we hit a button and the log history pops up so that we can see more than the last five things that have happened. Let's start with the Game Over state.

```lua
-- main.lua

State = {PLAYER_TURN = "1", ENEMY_TURN = "2", GAME_OVER = "3"}

...

function _update()
	if currentState == State.PLAYER_TURN then
		if player.hp <=0 then
			currentState = State.GAME_OVER
			return
		end
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
	elseif currentState == State.GAME_OVER then
		if btnp(5) then
			currentState = State.PLAYER_TURN
			entities = {}
			populateMap()
			logs = {}
			log("Welcome back to the dungeon!")
		end
	end
end

...

function drawInterface()
	clip()
	camera()
	drawSideUI()
	drawBottomUI()
	drawWindows()
end

function drawWindows()
	if currentState == State.GAME_OVER then
		drawGameOver()
	end
end

function drawGameOver()
	clip()
	camera()
	rectfill(20, 50, 460, 220, 1)
	rect(20, 50, 460, 220, 7)
	cursor(200, 80)
	print("You Died")
	print("Press X to restart")
end

```

So here, on the player's turn, we're checking to see if we've died. If so, we flip to the GAME_OVER state. That causes a big old window to be drawn on the screen, and let's us press X to restart our game. We reinitialize everything when restarting, and toss a cheeky "Welcome back" into the logs.

![[p7-game_over.gif]]

TODO: logs view
