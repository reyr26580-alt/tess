--[[ GEMINI AI ASSISTANT v2 - ONE LocalScript, pasted in 3 parts (1 → 2 → 3, in order).
     Chat-style UI. Add your Gemini API key in  Providers  (left sidebar). ]]

--// CONFIG
local DEFAULT_MODELS = { "gemini-2.5-flash", "gemini-flash-latest", "gemini-2.0-flash" }
local HISTORY_TURNS = 8
local DEFAULT_FLY_SPEED = 60
local KEY_FILE = "GeminiAssistantKey.txt"
local DATA_FILE = "GeminiAssistantData.json"
local LOGO = "✦" -- shows as a box on your device? change it to any symbol

--// SERVICES
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local HttpService = game:GetService("HttpService")
local PathfindingService = game:GetService("PathfindingService")
local UserInputService = game:GetService("UserInputService")
local TextChatService = game:GetService("TextChatService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local MarketplaceService = game:GetService("MarketplaceService")
local Workspace = game:GetService("Workspace")
local CoreGui = game:GetService("CoreGui")
local player = Players.LocalPlayer

local env = (getgenv and getgenv()) or _G
if env.GeminiAssistantUnload then pcall(env.GeminiAssistantUnload) end

-- hooks assigned later (Part 2 / 3)
local log, trace, refreshSend, refreshTools, renderChat, submit
local ERR, AI, SYS = "err", "ai", "sys"

--// API KEY
local API_KEY = ""
local function cleanKey(s)
	return (tostring(s or ""):gsub("[%s\"'`]", ""))
end
local function loadKey()
	local ok, v = pcall(function()
		if isfile and readfile and isfile(KEY_FILE) then return readfile(KEY_FILE) end
		return ""
	end)
	return ok and cleanKey(v) or ""
end
local function saveKey(k)
	pcall(function()
		if writefile then writefile(KEY_FILE, k) end
	end)
end
local function scrub(s) -- never show the key in error messages
	s = tostring(s)
	if #API_KEY >= 8 then
		local i, j = s:find(API_KEY, 1, true)
		while i do
			s = s:sub(1, i - 1) .. "[key]" .. s:sub(j + 1)
			i, j = s:find(API_KEY, 1, true)
		end
	end
	return s
end
API_KEY = loadKey()

--// THEME
local C = {
	bg = Color3.fromRGB(26, 26, 30), side = Color3.fromRGB(31, 31, 36),
	card = Color3.fromRGB(40, 40, 46), card2 = Color3.fromRGB(54, 54, 62),
	line = Color3.fromRGB(64, 64, 72), text = Color3.fromRGB(236, 236, 240), mute = Color3.fromRGB(150, 150, 162),
	red = Color3.fromRGB(255, 125, 125), green = Color3.fromRGB(120, 230, 150),
	amber = Color3.fromRGB(222, 158, 54), amberBg = Color3.fromRGB(58, 48, 36),
}
local ACCENTS = {
	Color3.fromRGB(76, 128, 245), Color3.fromRGB(155, 114, 203), Color3.fromRGB(217, 101, 121),
	Color3.fromRGB(60, 170, 110), Color3.fromRGB(230, 130, 60), Color3.fromRGB(40, 170, 190),
}

--// SETTINGS + CONVERSATIONS (saved to a file when your executor supports writefile)
local S = {
	model = DEFAULT_MODELS[1],
	models = { table.unpack(DEFAULT_MODELS) },
	temperature = 0.2,
	confirm = false,
	instructions = "",
	accent = 1,
	opacity = 0,
	textSize = 14,
	size = 2,
	waypoints = {},
	saved = {},
}
local convs, cur, convCounter = {}, nil, 0

local function newConv()
	convCounter += 1
	local c = { id = convCounter, title = "New chat", msgs = {}, history = {}, requests = 0 }
	table.insert(convs, 1, c)
	while #convs > 30 do table.remove(convs) end
	return c
end

local saveQueued = false
local function persist()
	if saveQueued then return end
	saveQueued = true
	task.delay(1.5, function()
		saveQueued = false
		pcall(function()
			if not writefile then return end
			local out = {}
			for _, c in ipairs(convs) do
				if #out < 15 and #c.msgs > 0 then
					local msgs, hist = {}, {}
					for j = math.max(1, #c.msgs - 59), #c.msgs do msgs[#msgs + 1] = c.msgs[j] end
					for j = math.max(1, #c.history - 15), #c.history do hist[#hist + 1] = c.history[j] end
					out[#out + 1] = { id = c.id, title = c.title, msgs = msgs, history = hist, requests = c.requests }
				end
			end
			writefile(DATA_FILE, HttpService:JSONEncode({ settings = S, convs = out, counter = convCounter }))
		end)
	end)
end

local function loadData()
	local ok, raw = pcall(function()
		if isfile and readfile and isfile(DATA_FILE) then return readfile(DATA_FILE) end
	end)
	if not ok or type(raw) ~= "string" or raw == "" then return end
	local ok2, data = pcall(function() return HttpService:JSONDecode(raw) end)
	if not ok2 or type(data) ~= "table" then return end
	if type(data.settings) == "table" then
		for k, v in pairs(data.settings) do
			if S[k] ~= nil and type(S[k]) == type(v) then S[k] = v end
		end
	end
	if type(data.convs) == "table" then
		for _, c in ipairs(data.convs) do
			if type(c) == "table" and type(c.msgs) == "table" then
				convs[#convs + 1] = {
					id = tonumber(c.id) or 0, title = tostring(c.title or "Chat"), msgs = c.msgs,
					history = type(c.history) == "table" and c.history or {}, requests = tonumber(c.requests) or 0,
				}
			end
		end
	end
	convCounter = tonumber(data.counter) or #convs
end
loadData()
if #S.models == 0 then S.models = { table.unpack(DEFAULT_MODELS) } end
local function hasModel(m)
	for _, x in ipairs(S.models) do
		if x == m then return true end
	end
	return false
end
if not hasModel(S.model) then S.model = S.models[1] end
cur = newConv()

--// STATE
local planId, runningPlan = 0, nil
local B = {}            -- active background behaviours: spin, follow, jump, orbit, spectate, fly, noclip
local flyObj = nil
local flySpeed = DEFAULT_FLY_SPEED
local busy = false

--// BASIC HELPERS
local function getHum()
	local c = player.Character
	return c and c:FindFirstChildOfClass("Humanoid")
end
local function getRoot()
	local c = player.Character
	return c and c:FindFirstChild("HumanoidRootPart")
end
local function flatDist(a, b)
	return Vector3.new(a.X - b.X, 0, a.Z - b.Z).Magnitude
end
local function alive(id) return id == planId end
local function sleep(t, id)
	local e = 0
	while e < t do
		if not alive(id) then return false end
		e += task.wait(math.min(0.1, t - e))
	end
	return alive(id)
end

--// BEHAVIOUR CONTROL
local function stopB(name)
	local rec = B[name]
	if rec then
		B[name] = nil
		if rec.cleanup then pcall(rec.cleanup) end
	end
end
local function startB(name, setup)
	stopB(name)
	local rec = {}
	B[name] = rec
	local ok, cleanup = pcall(setup)
	if ok then rec.cleanup = cleanup else B[name] = nil; log("⚠ " .. tostring(cleanup), ERR) end
	return rec
end
local function scheduleStop(name, rec, dur)
	task.delay(dur, function()
		if B[name] == rec then stopB(name) end
	end)
end
local function haltWalking()
	local h, r = getHum(), getRoot()
	if h and r then h:MoveTo(r.Position) end
end
local function stopBehaviors(includeModes)
	local names = {}
	for name in pairs(B) do
		if includeModes or (name ~= "fly" and name ~= "noclip") then names[#names + 1] = name end
	end
	for _, n in ipairs(names) do stopB(n) end
end
local function stopAll(includeModes)
	planId += 1
	stopBehaviors(includeModes)
	haltWalking()
end

--// TARGETING
local COLOR_ALIAS = { grey = "gray", silver = "gray", golden = "yellow", gold = "yellow", lime = "green", violet = "purple", magenta = "pink", teal = "cyan", aqua = "cyan", turquoise = "cyan", navy = "blue", tan = "brown" }

local function colorName(c)
	local h, s, v = c:ToHSV()
	h = h * 360
	if v < 0.18 then return "black" end
	if s < 0.15 then return (v > 0.85) and "white" or "gray" end
	if h < 15 or h >= 345 then return (v < 0.45) and "brown" or "red" end
	if h < 40 then return (v < 0.6) and "brown" or "orange" end
	if h < 70 then return "yellow" end
	if h < 165 then return "green" end
	if h < 200 then return "cyan" end
	if h < 260 then return "blue" end
	if h < 300 then return "purple" end
	return "pink"
end

local function isPlayerPart(p)
	local m = p.Parent
	if m and Players:GetPlayerFromCharacter(m) then return true end
	if m and m.Parent and Players:GetPlayerFromCharacter(m.Parent) then return true end
	return false
end

local function findPlayer(q)
	q = tostring(q or ""):lower()
	if q == "" then return nil end
	if q == "nearest" or q == "closest" or q == "random" then
		local r = getRoot()
		local list = {}
		for _, p in ipairs(Players:GetPlayers()) do
			local pr = p ~= player and p.Character and p.Character:FindFirstChild("HumanoidRootPart")
			if pr then list[#list + 1] = { p = p, d = r and (pr.Position - r.Position).Magnitude or 0 } end
		end
		if #list == 0 then return nil end
		if q == "random" then return list[math.random(1, #list)].p end
		table.sort(list, function(x, y) return x.d < y.d end)
		return list[1].p
	end
	local pre, sub
	for _, p in ipairs(Players:GetPlayers()) do
		if p ~= player then
			local n, d = p.Name:lower(), p.DisplayName:lower()
			if n == q or d == q then return p end
			if not pre and (n:sub(1, #q) == q or d:sub(1, #q) == q) then pre = p end
			if not sub and (n:find(q, 1, true) or d:find(q, 1, true)) then sub = p end
		end
	end
	return pre or sub
end

local function findWaypoint(q)
	q = tostring(q or ""):lower()
	local sub
	for name, w in pairs(S.waypoints) do
		local n = tostring(name):lower()
		if n == q then return { name = name, x = w.x, y = w.y, z = w.z } end
		if not sub and n:find(q, 1, true) then sub = { name = name, x = w.x, y = w.y, z = w.z } end
	end
	return sub
end

local function findPart(nameQ, colorQ)
	local root = getRoot()
	if not root then return nil end
	nameQ = (nameQ ~= nil and tostring(nameQ) ~= "") and tostring(nameQ):lower() or nil
	colorQ = (colorQ ~= nil and tostring(colorQ) ~= "") and tostring(colorQ):lower() or nil
	if colorQ then colorQ = COLOR_ALIAS[colorQ] or colorQ end
	if not nameQ and not colorQ then return nil end
	local best, bestD = nil, math.huge
	for _, p in ipairs(Workspace:GetDescendants()) do
		if p:IsA("BasePart") and not p:IsA("Terrain") and not isPlayerPart(p) then
			local ok = true
			if nameQ then
				ok = p.Name:lower():find(nameQ, 1, true) ~= nil
					or (p.Parent ~= nil and p.Parent ~= Workspace and p.Parent.Name:lower():find(nameQ, 1, true) ~= nil)
			end
			if ok and colorQ then
				if not nameQ and p.Transparency > 0.9 then ok = false
				else ok = colorName(p.Color) == colorQ end
			end
			if ok then
				local d = (p.Position - root.Position).Magnitude
				if d < bestD then best, bestD = p, d end
			end
		end
	end
	return best
end

local function topY(p)
	local c, s = p.CFrame, p.Size
	return p.Position.Y + (math.abs(c.RightVector.Y) * s.X + math.abs(c.UpVector.Y) * s.Y + math.abs(c.LookVector.Y) * s.Z) / 2
end

local function resolveTarget(a)
	if a.player ~= nil and tostring(a.player) ~= "" then
		local p = findPlayer(a.player)
		if not p then return nil, "Player '" .. tostring(a.player) .. "' not found" end
		return { player = p, label = p.Name }
	end
	if a.waypoint ~= nil and tostring(a.waypoint) ~= "" then
		local w = findWaypoint(a.waypoint)
		if not w then return nil, "No saved waypoint named '" .. tostring(a.waypoint) .. "'" end
		return { pos = Vector3.new(w.x, w.y, w.z), label = w.name }
	end
	local x, y, z = tonumber(a.x), tonumber(a.y), tonumber(a.z)
	if x and y and z then
		return { pos = Vector3.new(x, y, z), label = "(" .. x .. ", " .. y .. ", " .. z .. ")" }
	end
	if a.part_name ~= nil or a.color ~= nil then
		local part = findPart(a.part_name, a.color)
		if not part then
			return nil, "No part found (" .. tostring(a.color or "") .. " " .. tostring(a.part_name or "") .. ")"
		end
		return { part = part, label = part.Name }
	end
	return nil, "No target given"
end

local function goalOf(t)
	if t.player then
		local ch = t.player.Character
		local r = ch and ch:FindFirstChild("HumanoidRootPart")
		return r and r.Position
	elseif t.part then
		local p = t.part
		if not p.Parent then return nil end
		return Vector3.new(p.Position.X, topY(p) + 3, p.Position.Z)
	end
	return t.pos
end

local function partDistance(root, part)
	local lp = part.CFrame:PointToObjectSpace(root.Position)
	local h = part.Size / 2
	local c = Vector3.new(math.clamp(lp.X, -h.X, h.X), math.clamp(lp.Y, -h.Y, h.Y), math.clamp(lp.Z, -h.Z, h.Z))
	return (lp - c).Magnitude
end

local function arrived(root, t, goal, stop)
	if flatDist(root.Position, goal) <= stop and math.abs(root.Position.Y - goal.Y) <= 8 then return true end
	if t.part and t.part.Parent and partDistance(root, t.part) <= 3.6 then return true end
	return false
end

--// MOVEMENT
local function flyTo(t, id, speed, stopDist)
	local fo = flyObj
	if not fo then return false end
	speed = speed or flySpeed
	local t0 = os.clock()
	while alive(id) and fo.bv.Parent do
		local root, goal = getRoot(), goalOf(t)
		if not root or not goal then break end
		local delta = goal - root.Position
		local dist = delta.Magnitude
		if dist <= stopDist then
			fo.bv.Velocity = Vector3.zero
			return true
		end
		fo.bv.Velocity = delta.Unit * math.min(speed, dist * 6)
		if not B.spin then
			local flat = Vector3.new(delta.X, 0, delta.Z)
			if flat.Magnitude > 1 then root.CFrame = CFrame.lookAt(root.Position, root.Position + flat) end
		end
		if os.clock() - t0 > 120 then break end
		RunService.Heartbeat:Wait()
	end
	if fo.bv.Parent then fo.bv.Velocity = Vector3.zero end
	return false
end

local function getWaypoints(from, to)
	local path = PathfindingService:CreatePath({ AgentRadius = 2, AgentHeight = 5, AgentCanJump = true, AgentCanClimb = true })
	local ok = pcall(function() path:ComputeAsync(from, to) end)
	if ok and path.Status == Enum.PathStatus.Success then return path:GetWaypoints() end
	return nil
end

local function walkTo(t, id, stopDist)
	local hum, root = getHum(), getRoot()
	if not hum or not root or hum.Health <= 0 then return false end
	if flyObj then return flyTo(t, id, nil, stopDist or (t.player and 5 or 3)) end
	if hum.Sit then hum.Sit = false end
	stopDist = stopDist or (t.player and 5 or 3.5)
	local deadline = os.clock() + 90
	for _ = 1, 10 do
		if not alive(id) or os.clock() > deadline then return false end
		hum, root = getHum(), getRoot()
		local goal = goalOf(t)
		if not hum or not root or not goal or hum.Health <= 0 then return false end
		if arrived(root, t, goal, stopDist) then return true end

		local wps = getWaypoints(root.Position, goal)
		if wps then
			for i = 2, #wps do
				if not alive(id) then return false end
				local wp = wps[i]
				hum = getHum()
				if not hum then return false end
				if wp.Action == Enum.PathWaypointAction.Jump then hum.Jump = true end
				hum:MoveTo(wp.Position)
				local t0, sampleT, samplePos = os.clock(), os.clock(), root.Position
				local repath = false
				while alive(id) do
					root = getRoot()
					if not root then return false end
					if flatDist(root.Position, wp.Position) < 3 then break end
					local now = os.clock()
					if now - t0 > 6 then repath = true break end
					if now - sampleT > 1 then
						if (root.Position - samplePos).Magnitude < 1 then
							local h = getHum()
							if h then h.Jump = true; h:MoveTo(wp.Position) end
						end
						sampleT, samplePos = now, root.Position
					end
					if t.player then
						local g2 = goalOf(t)
						if g2 then
							if arrived(root, t, g2, stopDist) then return true end
							if (g2 - goal).Magnitude > 12 then repath = true break end
						end
					end
					task.wait(0.1)
				end
				if repath then break end
			end
		else
			local t0, sampleT, samplePos = os.clock(), os.clock(), root.Position
			while alive(id) and os.clock() - t0 < 8 do
				root = getRoot()
				if not root then return false end
				local g2 = goalOf(t) or goal
				if arrived(root, t, g2, stopDist) then return true end
				local h = getHum()
				if h then h:MoveTo(g2) end
				local now = os.clock()
				if now - sampleT > 1 then
					if h and (root.Position - samplePos).Magnitude < 1 then h.Jump = true end
					sampleT, samplePos = now, root.Position
				end
				task.wait(0.1)
			end
		end
		task.wait(0.1)
	end
	root = getRoot()
	local g = goalOf(t)
	return (root ~= nil and g ~= nil and arrived(root, t, g, stopDist)) or false
end

local function ensureFly()
	if flyObj and flyObj.bv.Parent then return true end
	local r = getRoot()
	if not r then return false end
	startB("fly", function()
		local bv = Instance.new("BodyVelocity")
		bv.MaxForce = Vector3.new(1e8, 1e8, 1e8)
		bv.Velocity = Vector3.zero
		bv.P = 1e4
		bv.Parent = r
		flyObj = { bv = bv }
		return function()
			flyObj = nil
			bv:Destroy()
		end
	end)
	return flyObj ~= nil
end

--// ACTIONS
local A = {}

A.walk_to = function(a, id)
	local t, err = resolveTarget(a)
	if not t then log("⚠ " .. err, ERR) return end
	local ok = walkTo(t, id, tonumber(a.distance))
	if not ok and alive(id) then log("⚠ Couldn't reach " .. t.label, ERR) end
end

A.teleport = function(a)
	local t, err = resolveTarget(a)
	if not t then log("⚠ " .. err, ERR) return end
	local r = getRoot()
	if not r then return end
	if t.player then
		local tr = t.player.Character and t.player.Character:FindFirstChild("HumanoidRootPart")
		if not tr then log("⚠ " .. t.label .. " has no character", ERR) return end
		r.CFrame = tr.CFrame * CFrame.new(0, 0, 4)
	else
		local g = goalOf(t)
		if not g then return end
		r.CFrame = CFrame.new(g) * r.CFrame.Rotation
	end
	r.AssemblyLinearVelocity = Vector3.zero
end

A.walk_direction = function(a, id)
	local r = getRoot()
	if not r then return end
	local look = Vector3.new(r.CFrame.LookVector.X, 0, r.CFrame.LookVector.Z)
	if look.Magnitude < 0.01 then look = Vector3.new(0, 0, -1) end
	look = look.Unit
	local right = look:Cross(Vector3.yAxis)
	local d = tostring(a.direction or "forward"):lower()
	local v = look
	if d == "back" or d == "backward" or d == "backwards" then v = -look
	elseif d == "left" then v = -right
	elseif d == "right" then v = right end
	local studs = math.clamp(tonumber(a.studs) or 10, 1, 1000)
	local ok = walkTo({ pos = r.Position + v * studs, label = "there" }, id, 2.5)
	if not ok and alive(id) then log("⚠ Got blocked while walking " .. d, ERR) end
end

A.follow = function(a)
	local p = findPlayer(a.player)
	if not p then log("⚠ Player '" .. tostring(a.player) .. "' not found", ERR) return end
	local dist = math.clamp(tonumber(a.distance) or 5, 2, 50)
	startB("follow", function()
		local running = true
		task.spawn(function()
			local sampleT, samplePos = os.clock(), nil
			while running do
				local h, r = getHum(), getRoot()
				local tr = p.Character and p.Character:FindFirstChild("HumanoidRootPart")
				if not p.Parent then
					running = false
					task.defer(stopB, "follow")
					break
				end
				if h and r and tr then
					if flyObj and flyObj.bv.Parent then
						local delta = tr.Position - r.Position
						flyObj.bv.Velocity = (delta.Magnitude > dist) and delta.Unit * math.min(flySpeed, delta.Magnitude * 4) or Vector3.zero
					else
						if h.Sit then h.Sit = false end
						local d = flatDist(r.Position, tr.Position)
						if d > dist then
							h:MoveTo(tr.Position)
							local now = os.clock()
							if samplePos == nil then samplePos = r.Position end
							if now - sampleT > 0.6 then
								if d > dist + 2 and (r.Position - samplePos).Magnitude < 0.6 then h.Jump = true end
								sampleT, samplePos = now, r.Position
							end
						else
							h:MoveTo(r.Position)
						end
					end
				end
				task.wait(0.15)
			end
		end)
		return function()
			running = false
			haltWalking()
		end
	end)
end

A.look_at = function(a)
	local t, err = resolveTarget(a)
	if not t then log("⚠ " .. err, ERR) return end
	local r, g = getRoot(), goalOf(t)
	if r and g and flatDist(r.Position, g) > 0.1 then
		r.CFrame = CFrame.lookAt(r.Position, Vector3.new(g.X, r.Position.Y, g.Z))
	end
end

A.jump = function(a, id)
	if not getHum() then return end
	local interval = math.max(0.15, tonumber(a.interval) or 0.7)
	local dur = tonumber(a.duration)
	if a.endless or dur then
		local rec = startB("jump", function()
			local running = true
			task.spawn(function()
				while running do
					local h = getHum()
					if h and h.Health > 0 then h.Jump = true end
					task.wait(interval)
				end
			end)
			return function() running = false end
		end)
		if dur then
			scheduleStop("jump", rec, dur)
			if not a.background then sleep(dur, id) end
		end
	else
		local n = math.clamp(math.floor(tonumber(a.count) or 1), 1, 100)
		for i = 1, n do
			if not alive(id) then return end
			local h = getHum()
			if h then h.Jump = true end
			if i < n then sleep(interval, id) end
		end
	end
end

A.sit = function(a)
	local h = getHum()
	if h then h.Sit = (a.state ~= false) end
end
A.stand = function() local h = getHum() if h then h.Sit = false end end

A.flip = function(a, id)
	local hum, root = getHum(), getRoot()
	if not hum or not root or flyObj then return end
	local dir = tostring(a.direction or "back"):lower()
	local rx, rz = 1, 0
	if dir == "front" or dir == "forward" then rx = -1
	elseif dir == "left" then rx, rz = 0, 1
	elseif dir == "right" then rx, rz = 0, -1 end
	local look = Vector3.new(root.CFrame.LookVector.X, 0, root.CFrame.LookVector.Z)
	if look.Magnitude < 0.01 then look = Vector3.new(0, 0, -1) end
	hum.Jump = true
	task.wait(0.06)
	root = getRoot()
	if not root then return end
	local v = root.AssemblyLinearVelocity
	root.AssemblyLinearVelocity = Vector3.new(v.X, 52, v.Z)
	hum.PlatformStand = true
	pcall(function()
		local dur, elapsed = 0.55, 0
		while elapsed < dur and alive(id) do
			local dt = RunService.Heartbeat:Wait()
			elapsed += dt
			local r = getRoot()
			if not r then break end
			local ang = (math.pi * 2) * dt / dur
			r.CFrame = r.CFrame * CFrame.Angles(ang * rx, 0, ang * rz)
		end
	end)
	root, hum = getRoot(), getHum()
	if root then root.CFrame = CFrame.lookAt(root.Position, root.Position + look) end
	if hum then
		hum.PlatformStand = false
		hum:ChangeState(Enum.HumanoidStateType.GettingUp)
	end
end

A.spin = function(a, id)
	local speed = math.clamp(tonumber(a.speed) or 720, 10, 5000)
	local sign = (tostring(a.direction):lower() == "left") and 1 or -1
	local dur = tonumber(a.duration)
	local rec = startB("spin", function()
		local h = getHum()
		if h then h.AutoRotate = false end
		local c = RunService.Heartbeat:Connect(function(dt)
			local r = getRoot()
			if r then r.CFrame = r.CFrame * CFrame.Angles(0, math.rad(speed) * dt * sign, 0) end
		end)
		return function()
			c:Disconnect()
			local h2 = getHum()
			if h2 then h2.AutoRotate = true end
		end
	end)
	if dur then
		scheduleStop("spin", rec, dur)
		if not a.background then sleep(dur, id) end
	end
end

A.orbit = function(a, id)
	local t, err = resolveTarget(a)
	if not t then log("⚠ " .. err, ERR) return end
	local radius = math.clamp(tonumber(a.radius) or 10, 3, 200)
	local speed = math.clamp(tonumber(a.speed) or 20, 2, 300)
	local dur = tonumber(a.duration)
	local rec = startB("orbit", function()
		local ang = 0
		local c = RunService.Heartbeat:Connect(function(dt)
			local r, center = getRoot(), goalOf(t)
			if not r or not center then return end
			ang += (speed / radius) * dt
			local pos = Vector3.new(center.X + math.cos(ang) * radius, center.Y, center.Z + math.sin(ang) * radius)
			local tangent = Vector3.new(-math.sin(ang), 0, math.cos(ang))
			r.CFrame = CFrame.lookAt(pos, pos + tangent)
			r.AssemblyLinearVelocity = Vector3.zero
		end)
		return function() c:Disconnect() end
	end)
	if dur then
		scheduleStop("orbit", rec, dur)
		if not a.background then sleep(dur, id) end
	end
end

A.spectate = function(a)
	local p = findPlayer(a.player)
	if not p then log("⚠ Player '" .. tostring(a.player) .. "' not found", ERR) return end
	startB("spectate", function()
		local c = RunService.RenderStepped:Connect(function()
			local cam = Workspace.CurrentCamera
			local h = p.Character and p.Character:FindFirstChildOfClass("Humanoid")
			if cam and h and cam.CameraSubject ~= h then cam.CameraSubject = h end
			if not p.Parent then task.defer(stopB, "spectate") end
		end)
		return function()
			c:Disconnect()
			local cam, own = Workspace.CurrentCamera, getHum()
			if cam and own then cam.CameraSubject = own end
		end
	end)
end

A.speed = function(a)
	local h, v = getHum(), tonumber(a.value)
	if h and v then h.WalkSpeed = math.clamp(v, 0, 1000) end
end

A.jump_power = function(a)
	local h, v = getHum(), tonumber(a.value)
	if not h or not v then return end
	v = math.clamp(v, 0, 1000)
	if h.UseJumpPower then h.JumpPower = v
	else h.JumpHeight = (v * v) / (2 * Workspace.Gravity) end
end

A.gravity = function(a)
	local v = tonumber(a.value)
	if v then Workspace.Gravity = math.clamp(v, 0, 1000) end
end

A.fly = function(a)
	if a.on == false then stopB("fly") return end
	flySpeed = math.clamp(tonumber(a.speed) or flySpeed, 5, 500)
	ensureFly()
end

A.fly_up = function(a, id)
	if not ensureFly() then return end
	local r = getRoot()
	if not r then return end
	flyTo({ pos = r.Position + Vector3.new(0, tonumber(a.studs) or 20, 0) }, id, nil, 1.5)
end

A.noclip = function(a)
	if a.on == false then stopB("noclip") return end
	startB("noclip", function()
		local saved = {}
		local c = RunService.Stepped:Connect(function()
			local ch = player.Character
			if not ch then return end
			for _, p in ipairs(ch:GetDescendants()) do
				if p:IsA("BasePart") then
					if saved[p] == nil then saved[p] = p.CanCollide end
					p.CanCollide = false
				end
			end
		end)
		return function()
			c:Disconnect()
			for p, v in pairs(saved) do
				if p.Parent then p.CanCollide = v end
			end
		end
	end)
end

A.save_waypoint = function(a)
	local r = getRoot()
	local name = tostring(a.name or ""):gsub("^%s+", ""):gsub("%s+$", "")
	if not r or name == "" then log("⚠ Give the waypoint a name", ERR) return end
	local function r1(n) return math.floor(n * 10 + 0.5) / 10 end
	S.waypoints[name] = { x = r1(r.Position.X), y = r1(r.Position.Y), z = r1(r.Position.Z) }
	persist()
	if refreshTools then refreshTools() end
	log("📍 Saved waypoint '" .. name .. "'", SYS)
end

A.delete_waypoint = function(a)
	local w = findWaypoint(a.name)
	if not w then log("⚠ No waypoint named '" .. tostring(a.name) .. "'", ERR) return end
	S.waypoints[w.name] = nil
	persist()
	if refreshTools then refreshTools() end
	log("🗑 Deleted waypoint '" .. w.name .. "'", SYS)
end

local function sendChat(text)
	pcall(function()
		if TextChatService.ChatVersion == Enum.ChatVersion.TextChatService then
			local ch = TextChatService:FindFirstChild("TextChannels")
			local general = ch and ch:FindFirstChild("RBXGeneral")
			if general then general:SendAsync(text) end
		else
			local ev = ReplicatedStorage:FindFirstChild("DefaultChatSystemChatEvents")
			local say = ev and ev:FindFirstChild("SayMessageRequest")
			if say then say:FireServer(text, "All") end
		end
	end)
end

A.emote = function(a)
	local name = tostring(a.name or "wave"):lower()
	local ch = player.Character
	local anim = ch and ch:FindFirstChild("Animate")
	local pe = anim and anim:FindFirstChild("PlayEmote")
	if pe and pe:IsA("BindableFunction") then
		pcall(function() pe:Invoke(name) end)
	else
		sendChat("/e " .. name)
	end
end

A.chat = function(a)
	if a.text then sendChat(tostring(a.text)) end
end

A.wait = function(a, id)
	sleep(math.clamp(tonumber(a.seconds) or 1, 0, 120), id)
end

A.stop = function(a)
	local w = tostring(a.what or "all"):lower()
	if w == "all" or w == "walk" or w == "movement" then
		stopBehaviors(false)
		haltWalking()
	else
		stopB(w)
	end
end

A.reset = function()
	local h = getHum()
	if h then h.Health = 0 end
end

A.go_to, A.walk, A.tp = A.walk_to, A.walk_to, A.teleport

local function runAction(a, id)
	local fn = A[tostring(a.type or ""):lower()]
	if not fn then log("⚠ Unknown action: " .. tostring(a.type), ERR) return end
	local ok, err = pcall(fn, a, id)
	if not ok then log("⚠ " .. tostring(a.type) .. " error: " .. tostring(err), ERR) end
end

local function runPlan(plan, id)
	for _, a in ipairs(plan.actions) do
		if not alive(id) then break end
		if type(a) == "table" then
			if a.together then task.spawn(runAction, a, id) else runAction(a, id) end
		end
	end
end

local function describe(a)
	local t = tostring(a.type or "?")
	local extra = a.player or a.waypoint or a.part_name or a.color or a.name or a.direction or a.text
		or a.seconds or a.duration or a.value or a.studs
	return extra ~= nil and (t .. "(" .. tostring(extra) .. ")") or t
end
--// GEMINI
local BASE_PROMPT = [==[
You are the brain of a Roblox character controller. The user types commands in plain language (any language) and you turn them into actions for THEIR OWN character.
Reply with ONE JSON object and nothing else:
{"say":"short friendly reply, max 1 sentence","actions":[ ...action objects... ]}

Actions run one after another. Walking, waiting and timed actions finish before the next one starts. Add "together":true to an action to start it and begin the next one at the same moment.

TARGET FIELDS (used by walk_to, teleport, look_at, orbit) - use only what is needed:
- "player": a player name. If the user's text resembles a name in the player list, use that exact name. You can also use "nearest", "closest" or "random".
- "waypoint": name of a saved waypoint (the [STATE] line lists them)
- "part_name": text contained in the name of a part/model in the map
- "color": red, orange, yellow, green, cyan, blue, purple, pink, brown, white, gray or black (color of a part). Can be combined with part_name.
- "x","y","z": world coordinates

ACTIONS
{"type":"walk_to", <target fields>, "distance": optional stop distance}   walks with pathfinding (flies if fly is on)
{"type":"teleport", <target fields>}
{"type":"walk_direction","direction":"forward|back|left|right","studs":10}
{"type":"follow","player":"Name","distance":5}      keeps following until stopped
{"type":"look_at", <target fields>}
{"type":"orbit", <target fields>,"radius":10,"speed":20,"duration":seconds}   circles around the target; omit duration = endless
{"type":"spectate","player":"Name"}                 watch through that player's camera until stopped
{"type":"jump","count":1,"interval":0.7}            add "endless":true, or "duration":seconds, to keep jumping
{"type":"sit","state":true}                         state false = stand up
{"type":"flip","direction":"back|front|left|right"}
{"type":"spin","speed":720,"direction":"left|right","duration":seconds}   omit duration = spin endlessly
{"type":"speed","value":16}   {"type":"jump_power","value":50}   {"type":"gravity","value":196.2}
{"type":"fly","on":true,"speed":60}   {"type":"fly_up","studs":30}   (negative studs = down)
{"type":"noclip","on":true}      (noclip without fly makes the character fall through the floor)
{"type":"save_waypoint","name":"home"}     saves the character's current position
{"type":"delete_waypoint","name":"home"}
{"type":"emote","name":"wave|point|dance|dance2|dance3|laugh|cheer"}
{"type":"chat","text":"..."}
{"type":"wait","seconds":2}
{"type":"stop","what":"all|spin|follow|jump|orbit|spectate|fly|noclip"}   "all" stops spin, follow, jump, orbit, spectate and the current walk. fly and noclip must be stopped by name.
{"type":"reset"}      respawn the character
Defaults: walkspeed 16, jump power 50, gravity 196.2.

RULES
1. Spin, jump or orbit with a "duration" blocks until it ends, unless "background":true. Endless ones never block.
2. "Do X while Y": start the endless thing, do the other action, then stop the endless thing (unless the user said it should keep going).
3. The [STATE] line tells you what is active right now, who is in the server and the saved waypoints. Use it (for example to stop what is running).
4. If the user asks for something these actions cannot do, explain briefly in "say" and return "actions":[].
5. Write "say" in the user's language. Never output anything outside the JSON object.
]==]

local function systemPrompt()
	local s = BASE_PROMPT
	if S.instructions ~= "" then
		s = s .. "\nEXTRA INSTRUCTIONS FROM THE USER (follow them, but never break the JSON format):\n" .. S.instructions .. "\n"
	end
	return s
end

local httpRequest = (syn and syn.request) or (http and http.request) or http_request or request or (fluxus and fluxus.request)

local function sendHttp(url, body)
	local req = { Url = url, Method = "POST", Headers = { ["Content-Type"] = "application/json" }, Body = body }
	if httpRequest then return httpRequest(req) end
	local r = HttpService:RequestAsync(req)
	return { StatusCode = r.StatusCode, Body = r.Body }
end

local function buildState()
	local r, h = getRoot(), getHum()
	local others = {}
	for _, p in ipairs(Players:GetPlayers()) do
		if p ~= player and #others < 60 then
			others[#others + 1] = (p.DisplayName ~= p.Name) and (p.Name .. " (" .. p.DisplayName .. ")") or p.Name
		end
	end
	local active = {}
	for n in pairs(B) do active[#active + 1] = n end
	table.sort(active)
	local wps = {}
	for n in pairs(S.waypoints) do wps[#wps + 1] = tostring(n) end
	table.sort(wps)
	local pos = r and (math.floor(r.Position.X + 0.5) .. ", " .. math.floor(r.Position.Y + 0.5) .. ", " .. math.floor(r.Position.Z + 0.5)) or "?"
	return "[STATE] my name: " .. player.Name .. " | position: " .. pos
		.. " | walkspeed: " .. (h and math.floor(h.WalkSpeed) or "?")
		.. " | active behaviours: " .. (#active > 0 and table.concat(active, ", ") or "none")
		.. " | waypoints: " .. (#wps > 0 and table.concat(wps, ", ") or "none")
		.. " | other players: " .. (#others > 0 and table.concat(others, ", ") or "none")
end

-- returns text, err, kind ("fatal" = stop trying other models), milliseconds
local function geminiRequest(model, contents, opts)
	opts = opts or {}
	local gen = { temperature = S.temperature, maxOutputTokens = 4096 }
	if opts.json ~= false then gen.responseMimeType = "application/json" end
	local payload = { contents = contents, generationConfig = gen }
	if opts.system then payload.systemInstruction = { parts = { { text = opts.system } } } end
	local url = "https://generativelanguage.googleapis.com/v1beta/models/" .. model .. ":generateContent?key=" .. API_KEY
	local t0 = os.clock()
	local ok, res = pcall(sendHttp, url, HttpService:JSONEncode(payload))
	local ms = math.floor((os.clock() - t0) * 1000)
	if not ok then
		return nil, "HTTP request failed: " .. scrub(res) .. " (your executor needs a working request function)", "fatal", ms
	end
	local code = tonumber(res.StatusCode) or 0
	local data
	pcall(function() data = HttpService:JSONDecode(res.Body) end)
	if code == 200 and type(data) == "table" then
		local cand = data.candidates and data.candidates[1]
		local parts = cand and cand.content and cand.content.parts
		if parts then
			local out = {}
			for _, p in ipairs(parts) do
				if p.text and not p.thought then out[#out + 1] = p.text end
			end
			local text = table.concat(out)
			if text ~= "" then return text, nil, nil, ms end
		end
		local why = (data.promptFeedback and data.promptFeedback.blockReason) or (cand and cand.finishReason) or "empty reply"
		return nil, model .. ": " .. tostring(why), "retry", ms
	end
	local msg = (type(data) == "table" and data.error and data.error.message) or ("HTTP " .. code)
	if tostring(msg):lower():find("api key") then
		return nil, "API key problem - " .. scrub(msg) .. " (open Providers and paste a new key)", "fatal", ms
	end
	return nil, model .. ": " .. scrub(msg), "retry", ms
end

-- returns text, err, modelUsed, ms
local function askGemini(conv, userText)
	if API_KEY == "" then return nil, "No API key yet. Open Providers, paste your Gemini key and tap Save." end
	local contents = {}
	for _, h in ipairs(conv.history) do contents[#contents + 1] = h end
	contents[#contents + 1] = { role = "user", parts = { { text = buildState() .. "\n\n[USER COMMAND]\n" .. userText } } }
	local order = { S.model }
	for _, m in ipairs(S.models) do
		if m ~= S.model then order[#order + 1] = m end
	end
	local lastErr = "No response."
	for _, model in ipairs(order) do
		local text, err, kind, ms = geminiRequest(model, contents, { system = systemPrompt() })
		if text then
			conv.history[#conv.history + 1] = { role = "user", parts = { { text = userText } } }
			conv.history[#conv.history + 1] = { role = "model", parts = { { text = text } } }
			while #conv.history > HISTORY_TURNS * 2 do table.remove(conv.history, 1) end
			return text, nil, model, ms
		end
		lastErr = err
		if kind == "fatal" then return nil, err end
	end
	return nil, lastErr
end

local function parsePlan(raw)
	raw = tostring(raw):gsub("^%s*```%w*%s*", ""):gsub("%s*```%s*$", "")
	local ok, data = pcall(function() return HttpService:JSONDecode(raw) end)
	if not ok then
		local s = raw:match("%b{}")
		if s then ok, data = pcall(function() return HttpService:JSONDecode(s) end) end
	end
	if not ok or type(data) ~= "table" then return nil end
	if data.actions == nil and data[1] ~= nil then data = { actions = data } end
	if type(data.actions) ~= "table" then data.actions = {} end
	return data
end

--// UI FRAMEWORK
local function make(class, props, parent)
	local o = Instance.new(class)
	for k, v in pairs(props) do o[k] = v end
	o.Parent = parent
	return o
end
local function round(o, r) make("UICorner", { CornerRadius = UDim.new(0, r or 8) }, o) end
local function stroke(o, color, th)
	return make("UIStroke", { Color = color or C.line, Thickness = th or 1, ApplyStrokeMode = Enum.ApplyStrokeMode.Border }, o)
end
local function pad(o, l, t, r, b)
	make("UIPadding", { PaddingLeft = UDim.new(0, l), PaddingTop = UDim.new(0, t or l), PaddingRight = UDim.new(0, r or l), PaddingBottom = UDim.new(0, b or t or l) }, o)
end
local orderCounter = 0
local function nextOrder() orderCounter += 1; return orderCounter end
local function clearChildren(f)
	for _, c in ipairs(f:GetChildren()) do
		if c:IsA("GuiObject") then c:Destroy() end
	end
end

local accentRefs = {}
local function accentOf() return ACCENTS[S.accent] or ACCENTS[1] end
local function reg(obj, prop)
	accentRefs[#accentRefs + 1] = { obj, prop }
	obj[prop] = accentOf()
end
local function applyAccent()
	for _, r in ipairs(accentRefs) do
		pcall(function() r[1][r[2]] = accentOf() end)
	end
end

local function logo(parent, size)
	local l = make("TextLabel", { Size = UDim2.fromOffset(size, size), BackgroundTransparency = 1, Text = LOGO, TextColor3 = Color3.new(1, 1, 1), Font = Enum.Font.GothamBold, TextSize = size, LayoutOrder = nextOrder() }, parent)
	make("UIGradient", { Rotation = 45, Color = ColorSequence.new({
		ColorSequenceKeypoint.new(0, Color3.fromRGB(66, 133, 244)),
		ColorSequenceKeypoint.new(0.55, Color3.fromRGB(155, 114, 203)),
		ColorSequenceKeypoint.new(1, Color3.fromRGB(217, 101, 121)),
	}) }, l)
	return l
end

local function label(parent, text, o)
	o = o or {}
	return make("TextLabel", { Size = UDim2.new(1, 0, 0, 0), AutomaticSize = Enum.AutomaticSize.Y, BackgroundTransparency = 1, Text = text, TextWrapped = true, TextXAlignment = Enum.TextXAlignment.Left, TextYAlignment = Enum.TextYAlignment.Top, Font = o.bold and Enum.Font.GothamBold or Enum.Font.Gotham, TextSize = o.size or 13, TextColor3 = o.color or C.text, LayoutOrder = o.order or nextOrder() }, parent)
end

local function btn(parent, text, cb, o)
	o = o or {}
	local b = make("TextButton", { Size = UDim2.fromOffset(o.w or 90, o.h or 30), BackgroundColor3 = o.color or C.card2, Text = text, TextColor3 = o.textColor or C.text, Font = Enum.Font.GothamMedium, TextSize = o.size or 13, LayoutOrder = o.order or nextOrder() }, parent)
	round(b, 8)
	if o.accent then reg(b, "BackgroundColor3") end
	if cb then b.Activated:Connect(cb) end
	return b
end

local function row(parent, h)
	local r = make("Frame", { Size = UDim2.new(1, 0, 0, h or 30), BackgroundTransparency = 1, LayoutOrder = nextOrder() }, parent)
	make("UIListLayout", { FillDirection = Enum.FillDirection.Horizontal, SortOrder = Enum.SortOrder.LayoutOrder, Padding = UDim.new(0, 6), VerticalAlignment = Enum.VerticalAlignment.Center }, r)
	return r
end

local function textbox(parent, placeholder, text)
	local t = make("TextBox", { Size = UDim2.new(1, 0, 0, 32), BackgroundColor3 = C.bg, BorderSizePixel = 0, Text = text or "", PlaceholderText = placeholder, PlaceholderColor3 = C.mute, TextColor3 = C.text, Font = Enum.Font.Gotham, TextSize = 13, ClearTextOnFocus = false, ClipsDescendants = true, TextXAlignment = Enum.TextXAlignment.Left, LayoutOrder = nextOrder() }, parent)
	round(t, 8); stroke(t); pad(t, 8, 0)
	return t
end

local function section(parent, title)
	local card = make("Frame", { Size = UDim2.new(1, 0, 0, 0), AutomaticSize = Enum.AutomaticSize.Y, BackgroundColor3 = C.card, BorderSizePixel = 0, LayoutOrder = nextOrder() }, parent)
	round(card, 10); stroke(card); pad(card, 10)
	make("UIListLayout", { SortOrder = Enum.SortOrder.LayoutOrder, Padding = UDim.new(0, 6) }, card)
	if title then label(card, title, { bold = true, size = 14 }) end
	return card
end

-- "name   value   [-] [+]" row. fmt() returns the shown value, onStep(dir) changes it.
local function stepper(parent, name, fmt, onStep)
	local r = row(parent, 30)
	make("TextLabel", { Size = UDim2.new(1, -126, 1, 0), BackgroundTransparency = 1, Text = name, TextColor3 = C.text, Font = Enum.Font.Gotham, TextSize = 13, TextXAlignment = Enum.TextXAlignment.Left, TextTruncate = Enum.TextTruncate.AtEnd, LayoutOrder = nextOrder() }, r)
	local val = make("TextLabel", { Size = UDim2.fromOffset(40, 30), BackgroundTransparency = 1, Text = fmt(), TextColor3 = C.mute, Font = Enum.Font.GothamMedium, TextSize = 13, LayoutOrder = nextOrder() }, r)
	btn(r, "-", function() onStep(-1); val.Text = fmt() end, { w = 32 })
	btn(r, "+", function() onStep(1); val.Text = fmt() end, { w = 32 })
	return function() val.Text = fmt() end
end

local function viewport()
	local cam = Workspace.CurrentCamera
	return cam and cam.ViewportSize or Vector2.new(800, 400)
end
local function windowSize()
	local vp = viewport()
	local f = ({ 0.82, 1, 1.2 })[S.size] or 1
	local w = math.clamp(math.floor(440 * f), 300, math.max(300, vp.X - 16))
	local h = math.clamp(math.floor(272 * f), 220, math.max(220, vp.Y - 30))
	return w, h
end

--// WINDOW
local SBW = 136
local conns = {}
local gui = make("ScreenGui", { Name = "GeminiAssistant", ResetOnSpawn = false, DisplayOrder = 100, IgnoreGuiInset = true, ZIndexBehavior = Enum.ZIndexBehavior.Sibling })
local main = make("Frame", { Name = "Main", BackgroundColor3 = C.bg, BorderSizePixel = 0, ClipsDescendants = true, Active = true }, gui)
round(main, 14); stroke(main)

local side = make("Frame", { Name = "Sidebar", Size = UDim2.new(0, SBW, 1, 0), BackgroundColor3 = C.side, BorderSizePixel = 0 }, main)
make("Frame", { Size = UDim2.new(0, 1, 1, 0), Position = UDim2.new(1, -1, 0, 0), BackgroundColor3 = C.line, BorderSizePixel = 0 }, side)
local right = make("Frame", { Name = "Right", Position = UDim2.new(0, SBW, 0, 0), Size = UDim2.new(1, -SBW, 1, 0), BackgroundColor3 = C.bg, BorderSizePixel = 0 }, main)

-- sidebar: icon row
local function iconBtn(parent, text)
	local b = make("TextButton", { Size = UDim2.fromOffset(28, 28), BackgroundTransparency = 1, Text = text, TextColor3 = C.mute, Font = Enum.Font.GothamMedium, TextSize = 16, LayoutOrder = nextOrder() }, parent)
	round(b, 6)
	return b
end
local iconRow = make("Frame", { Position = UDim2.new(0, 6, 0, 6), Size = UDim2.new(1, -12, 0, 28), BackgroundTransparency = 1 }, side)
make("UIListLayout", { FillDirection = Enum.FillDirection.Horizontal, SortOrder = Enum.SortOrder.LayoutOrder, Padding = UDim.new(0, 4) }, iconRow)
local sbToggle = iconBtn(iconRow, "☰")
local sbSearch = iconBtn(iconRow, "🔍")
local sbBack = iconBtn(iconRow, "←")
local sbFwd = iconBtn(iconRow, "→")

-- sidebar: tabs
local tabsRow = make("Frame", { Position = UDim2.new(0, 6, 0, 38), Size = UDim2.new(1, -12, 0, 26), BackgroundTransparency = 1 }, side)
make("UIListLayout", { FillDirection = Enum.FillDirection.Horizontal, SortOrder = Enum.SortOrder.LayoutOrder, Padding = UDim.new(0, 3) }, tabsRow)
local tabBtns = {}
for _, t in ipairs({ { "chat", "Chat" }, { "tools", "Tools" }, { "logs", "Logs" } }) do
	local b = make("TextButton", { Size = UDim2.new(1 / 3, -2, 1, 0), BackgroundColor3 = C.side, Text = t[2], TextColor3 = C.mute, Font = Enum.Font.GothamMedium, TextSize = 12, LayoutOrder = nextOrder() }, tabsRow)
	round(b, 7)
	tabBtns[t[1]] = b
end

local newBtn = make("TextButton", { Position = UDim2.new(0, 6, 0, 68), Size = UDim2.new(1, -12, 0, 28), BackgroundTransparency = 1, Text = "+  New conversation", TextColor3 = C.text, Font = Enum.Font.GothamMedium, TextSize = 12 }, side)
round(newBtn, 8); stroke(newBtn)

-- sidebar: navigation list
local navScroll = make("ScrollingFrame", { Position = UDim2.new(0, 6, 0, 100), Size = UDim2.new(1, -12, 1, -158), BackgroundTransparency = 1, BorderSizePixel = 0, ScrollBarThickness = 2, CanvasSize = UDim2.new(), AutomaticCanvasSize = Enum.AutomaticSize.Y }, side)
make("UIListLayout", { SortOrder = Enum.SortOrder.LayoutOrder, Padding = UDim.new(0, 2) }, navScroll)
local navBtns = {}
for _, it in ipairs({
	{ "uimods", "🎨  UI mods" }, { "settings", "⚙  Settings" }, { "providers", "🔌  Providers" }, { "tools", "🧰  Tools" },
	{ "saved", "⭐  Saved commands" }, { "logs", "📜  Logs & traces" }, { "convs", "💬  Conversations" },
}) do
	local b = make("TextButton", { Size = UDim2.new(1, 0, 0, 28), BackgroundTransparency = 1, Text = it[2], TextColor3 = C.text, Font = Enum.Font.Gotham, TextSize = 13, TextXAlignment = Enum.TextXAlignment.Left, TextTruncate = Enum.TextTruncate.AtEnd, LayoutOrder = nextOrder() }, navScroll)
	round(b, 7); pad(b, 8, 0)
	navBtns[it[1]] = b
end

-- sidebar: user card
local userCard = make("Frame", { Position = UDim2.new(0, 6, 1, -52), Size = UDim2.new(1, -12, 0, 46), BackgroundColor3 = C.card, BorderSizePixel = 0 }, side)
round(userCard, 10); stroke(userCard)
local avatar = make("ImageLabel", { Position = UDim2.new(0, 6, 0.5, -17), Size = UDim2.fromOffset(34, 34), BackgroundColor3 = C.card2, BorderSizePixel = 0, Image = "" }, userCard)
round(avatar, 17)
local cardName = make("TextLabel", { Position = UDim2.new(0, 46, 0, 6), Size = UDim2.new(1, -52, 0, 16), BackgroundTransparency = 1, Text = player.DisplayName, TextColor3 = C.text, Font = Enum.Font.GothamBold, TextSize = 12, TextXAlignment = Enum.TextXAlignment.Left, TextTruncate = Enum.TextTruncate.AtEnd }, userCard)
local cardSub = make("TextLabel", { Position = UDim2.new(0, 46, 0, 23), Size = UDim2.new(1, -52, 0, 14), BackgroundTransparency = 1, Text = "", TextColor3 = C.mute, Font = Enum.Font.Gotham, TextSize = 11, TextXAlignment = Enum.TextXAlignment.Left, TextTruncate = Enum.TextTruncate.AtEnd }, userCard)
local cardBtn = make("TextButton", { Size = UDim2.new(1, 0, 1, 0), BackgroundTransparency = 1, Text = "" }, userCard)

-- right: header
local header = make("Frame", { Size = UDim2.new(1, 0, 0, 44), BackgroundTransparency = 1 }, right)
make("Frame", { Size = UDim2.new(1, 0, 0, 1), Position = UDim2.new(0, 0, 1, -1), BackgroundColor3 = C.line, BorderSizePixel = 0 }, header)
local hdrToggle = iconBtn(header, "☰")
hdrToggle.Position = UDim2.fromOffset(6, 8)
local hdrLogo = logo(header, 20)
hdrLogo.Position = UDim2.fromOffset(40, 12)
local titleLbl = make("TextLabel", { Position = UDim2.new(0, 66, 0, 5), Size = UDim2.new(1, -164, 0, 18), BackgroundTransparency = 1, Text = "", TextColor3 = C.text, Font = Enum.Font.GothamBold, TextSize = 14, TextXAlignment = Enum.TextXAlignment.Left, TextTruncate = Enum.TextTruncate.AtEnd }, header)
local subLbl = make("TextLabel", { Position = UDim2.new(0, 66, 0, 24), Size = UDim2.new(1, -164, 0, 14), BackgroundTransparency = 1, Text = "", TextColor3 = C.mute, Font = Enum.Font.Gotham, TextSize = 11, TextXAlignment = Enum.TextXAlignment.Left, TextTruncate = Enum.TextTruncate.AtEnd }, header)
local hdrBtns = make("Frame", { Position = UDim2.new(1, -94, 0, 8), Size = UDim2.fromOffset(90, 28), BackgroundTransparency = 1 }, header)
make("UIListLayout", { FillDirection = Enum.FillDirection.Horizontal, SortOrder = Enum.SortOrder.LayoutOrder, Padding = UDim.new(0, 3) }, hdrBtns)
local hdrMin, hdrMax, hdrClose = iconBtn(hdrBtns, "–"), iconBtn(hdrBtns, "□"), iconBtn(hdrBtns, "✕")

local pageHost = make("Frame", { Position = UDim2.new(0, 0, 0, 45), Size = UDim2.new(1, 0, 1, -45), BackgroundTransparency = 1 }, right)
local pages, currentPage = {}, "chat"

local function newPage(name)
	local p = make("Frame", { Name = name, Size = UDim2.new(1, 0, 1, 0), BackgroundTransparency = 1, Visible = false }, pageHost)
	pages[name] = p
	return p
end
local function scrollPage(name)
	local p = newPage(name)
	local sc = make("ScrollingFrame", { Position = UDim2.new(0, 6, 0, 4), Size = UDim2.new(1, -12, 1, -8), BackgroundTransparency = 1, BorderSizePixel = 0, ScrollBarThickness = 3, CanvasSize = UDim2.new(), AutomaticCanvasSize = Enum.AutomaticSize.Y }, p)
	make("UIListLayout", { SortOrder = Enum.SortOrder.LayoutOrder, Padding = UDim.new(0, 8) }, sc)
	pad(sc, 4, 4, 4, 10)
	return p, sc
end

-- chat page (messages + input bar)
local chatPage = make("Frame", { Name = "chat", Size = UDim2.new(1, 0, 1, 0), BackgroundTransparency = 1 }, pageHost)
pages.chat = chatPage
local chatScroll = make("ScrollingFrame", { Position = UDim2.new(0, 8, 0, 4), Size = UDim2.new(1, -16, 1, -58), BackgroundTransparency = 1, BorderSizePixel = 0, ScrollBarThickness = 3, CanvasSize = UDim2.new(), AutomaticCanvasSize = Enum.AutomaticSize.Y }, chatPage)
make("UIListLayout", { SortOrder = Enum.SortOrder.LayoutOrder, Padding = UDim.new(0, 8) }, chatScroll)
pad(chatScroll, 2, 2, 6, 8)

local inputBar = make("Frame", { Position = UDim2.new(0, 8, 1, -50), Size = UDim2.new(1, -16, 0, 42), BackgroundColor3 = C.card, BorderSizePixel = 0 }, chatPage)
round(inputBar, 14); stroke(inputBar)
local plusBtn = make("TextButton", { Position = UDim2.new(0, 4, 0.5, -16), Size = UDim2.fromOffset(32, 32), BackgroundTransparency = 1, Text = "+", TextColor3 = C.text, Font = Enum.Font.GothamMedium, TextSize = 22 }, inputBar)
local inputBox = make("TextBox", { Position = UDim2.new(0, 38, 0, 0), Size = UDim2.new(1, -112, 1, 0), BackgroundTransparency = 1, Text = "", PlaceholderText = "Tell me what to do…", PlaceholderColor3 = C.mute, TextColor3 = C.text, Font = Enum.Font.Gotham, TextSize = 14, ClearTextOnFocus = false, ClipsDescendants = true, TextXAlignment = Enum.TextXAlignment.Left }, inputBar)
local dotsBtn = make("TextButton", { Position = UDim2.new(1, -72, 0.5, -16), Size = UDim2.fromOffset(30, 32), BackgroundTransparency = 1, Text = "…", TextColor3 = C.mute, Font = Enum.Font.GothamBold, TextSize = 18 }, inputBar)
local sendBtn = make("TextButton", { Position = UDim2.new(1, -38, 0.5, -16), Size = UDim2.fromOffset(32, 32), Text = "↑", TextColor3 = Color3.new(1, 1, 1), Font = Enum.Font.GothamBold, TextSize = 17 }, inputBar)
round(sendBtn, 16)
reg(sendBtn, "BackgroundColor3")

-- popup menu + toast + minimized pill
local menuBlocker
local function closeMenu()
	if menuBlocker then menuBlocker:Destroy(); menuBlocker = nil end
end
local function showMenu(anchor, items, alignRight)
	closeMenu()
	local blocker = make("TextButton", { Size = UDim2.new(1, 0, 1, 0), BackgroundTransparency = 1, Text = "", AutoButtonColor = false, ZIndex = 50 }, main)
	menuBlocker = blocker
	local w, h = 214, #items * 30 + 10
	local box = make("Frame", { Size = UDim2.fromOffset(w, h), BackgroundColor3 = C.card, BorderSizePixel = 0, Active = true, ZIndex = 51 }, blocker)
	round(box, 10); stroke(box)
	local ap = anchor.AbsolutePosition - main.AbsolutePosition
	local x = alignRight and (ap.X + anchor.AbsoluteSize.X - w) or ap.X
	x = math.clamp(x, 6, math.max(6, main.AbsoluteSize.X - w - 6))
	local y = ap.Y - h - 6
	if y < 6 then y = ap.Y + anchor.AbsoluteSize.Y + 6 end
	box.Position = UDim2.fromOffset(x, y)
	for i, it in ipairs(items) do
		local b = make("TextButton", { Position = UDim2.fromOffset(5, 5 + (i - 1) * 30), Size = UDim2.new(1, -10, 0, 28), BackgroundTransparency = 1, Text = it[1], TextColor3 = C.text, Font = Enum.Font.Gotham, TextSize = 13, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 52 }, box)
		round(b, 6); pad(b, 8, 0)
		b.Activated:Connect(function()
			closeMenu()
			it[2]()
		end)
	end
	blocker.Activated:Connect(closeMenu)
end

local function toast(text)
	local t = make("TextLabel", { Size = UDim2.new(0, 0, 0, 28), AutomaticSize = Enum.AutomaticSize.X, AnchorPoint = Vector2.new(0.5, 1), Position = UDim2.new(0.5, 0, 1, -62), BackgroundColor3 = C.card2, Text = "  " .. text .. "  ", TextColor3 = C.text, Font = Enum.Font.GothamMedium, TextSize = 12, ZIndex = 60 }, main)
	round(t, 14); stroke(t)
	task.delay(2, function()
		if t.Parent then t:Destroy() end
	end)
end

local mini = make("TextButton", { Size = UDim2.fromOffset(46, 46), BackgroundColor3 = C.card, Text = "", Visible = false, Active = true, ZIndex = 100 }, gui)
round(mini, 23); stroke(mini, C.line, 1.5)
local miniLogo = logo(mini, 26)
miniLogo.AnchorPoint = Vector2.new(0.5, 0.5)
miniLogo.Position = UDim2.new(0.5, 0, 0.5, 0)
miniLogo.Active = false

local function mount(g)
	local ok = pcall(function() g.Parent = (gethui and gethui()) or CoreGui end)
	if not ok or not g.Parent then g.Parent = player:WaitForChild("PlayerGui") end
end
