# Proton-reanimation
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local player = Players.LocalPlayer

local active = true
local reanimated = false
local metaHooked = false

local VOID_Y = workspace.FallenPartsDestroyHeight + 25
local lastSafeCF = nil

-------------------------------------------------
-- MOTOR6D SYSTEM
-------------------------------------------------
local function ensureMotor6D(parent, name, part0, part1, c0, c1)
	if not parent or not part0 or not part1 then return nil end
	local existing = parent:FindFirstChild(name)
	if existing and existing:IsA("Motor6D") then
		existing.Part0 = part0
		existing.Part1 = part1
		if c0 then existing.C0 = c0 end
		if c1 then existing.C1 = c1 end
		existing.Enabled = true
		return existing
	end
	local m = Instance.new("Motor6D")
	m.Name = name
	m.Part0 = part0
	m.Part1 = part1
	if c0 then m.C0 = c0 end
	if c1 then m.C1 = c1 end
	m.Parent = parent
	return m
end

local function setupMotor6Ds(char)
	local root = char:FindFirstChild("HumanoidRootPart")
	local torso = char:FindFirstChild("Torso") or char:FindFirstChild("UpperTorso")
	local head = char:FindFirstChild("Head")
	if not root or not torso then return end

	-- RootJoint (HumanoidRootPart ↔ Torso)
	ensureMotor6D(
		root, "RootJoint",
		root, torso,
		CFrame.new(0, 0, 0) * CFrame.Angles(-math.pi / 2, 0, math.pi),
		CFrame.new(0, 0, 0) * CFrame.Angles(-math.pi / 2, 0, math.pi)
	)

	-- Neck (Torso ↔ Head)
	if head then
		ensureMotor6D(
			torso, "Neck",
			torso, head,
			CFrame.new(0, 1, 0) * CFrame.Angles(-math.pi / 2, 0, math.pi),
			CFrame.new(0, -0.5, 0) * CFrame.Angles(-math.pi / 2, 0, math.pi)
		)
	end

	-- R6 limbs
	local ra = char:FindFirstChild("Right Arm")
	local la = char:FindFirstChild("Left Arm")
	local rl = char:FindFirstChild("Right Leg")
	local ll = char:FindFirstChild("Left Leg")

	if ra then
		ensureMotor6D(torso, "Right Shoulder", torso, ra,
			CFrame.new(1, 0.5, 0) * CFrame.Angles(0, math.pi / 2, 0),
			CFrame.new(-0.5, 0.5, 0) * CFrame.Angles(0, math.pi / 2, 0))
	end
	if la then
		ensureMotor6D(torso, "Left Shoulder", torso, la,
			CFrame.new(-1, 0.5, 0) * CFrame.Angles(0, -math.pi / 2, 0),
			CFrame.new(0.5, 0.5, 0) * CFrame.Angles(0, -math.pi / 2, 0))
	end
	if rl then
		ensureMotor6D(torso, "Right Hip", torso, rl,
			CFrame.new(0.5, -1, 0) * CFrame.Angles(0, math.pi / 2, 0),
			CFrame.new(0, 1, 0) * CFrame.Angles(0, math.pi / 2, 0))
	end
	if ll then
		ensureMotor6D(torso, "Left Hip", torso, ll,
			CFrame.new(-0.5, -1, 0) * CFrame.Angles(0, -math.pi / 2, 0),
			CFrame.new(0, 1, 0) * CFrame.Angles(0, -math.pi / 2, 0))
	end

	-- R15 extra
	local lower = char:FindFirstChild("LowerTorso")
	local upper = char:FindFirstChild("UpperTorso")
	if lower and upper and root then
		ensureMotor6D(root, "Root", root, lower, CFrame.new(0, 0, 0), CFrame.new(0, 0, 0))
		ensureMotor6D(lower, "Waist", lower, upper, CFrame.new(0, 0.8, 0), CFrame.new(0, -0.2, 0))
	end

	-- Keep Motor6Ds alive (anti-break)
	for _, v in ipairs(char:GetDescendants()) do
		if v:IsA("Motor6D") then
			v:GetPropertyChangedSignal("Part0"):Connect(function()
				if active and (not v.Part0 or not v.Part1) then
					task.defer(function()
						if char and char.Parent then
							setupMotor6Ds(char)
						end
					end)
				end
			end)
		end
	end
end

-------------------------------------------------
-- ANTI-FALL
-------------------------------------------------
local function antiFall(char)
	local root = char:FindFirstChild("HumanoidRootPart")
	if not root then return end

	for _, name in ipairs({"HumanoidRootPart", "Torso", "UpperTorso", "LowerTorso", "Head"}) do
		local p = char:FindFirstChild(name)
		if p and p:IsA("BasePart") then
			p.CanCollide = true
		end
	end

	if root.Position.Y > VOID_Y + 10 then
		lastSafeCF = root.CFrame
	end

	if root.Position.Y < VOID_Y then
		if lastSafeCF then
			root.CFrame = lastSafeCF + Vector3.new(0, 5, 0)
		else
			root.CFrame = CFrame.new(0, 50, 0)
		end
		root.AssemblyLinearVelocity = Vector3.zero
		root.AssemblyAngularVelocity = Vector3.zero
	end
end

-------------------------------------------------
-- MAX GODMODE
-------------------------------------------------
local function maxGodmode(char)
	local hum = char:FindFirstChildOfClass("Humanoid")
	local root = char:FindFirstChild("HumanoidRootPart")
	if not hum then return end

	hum.MaxHealth = 1e9
	hum.Health = 1e9
	hum.BreakJointsOnDeath = false
	hum.RequiresNeck = false

	setupMotor6Ds(char)

	for _, state in ipairs(Enum.HumanoidStateType:GetEnumItems()) do
		if state ~= Enum.HumanoidStateType.Running
			and state ~= Enum.HumanoidStateType.RunningNoPhysics
			and state ~= Enum.HumanoidStateType.Jumping
			and state ~= Enum.HumanoidStateType.Freefall
			and state ~= Enum.HumanoidStateType.Landed
			and state ~= Enum.HumanoidStateType.Climbing
			and state ~= Enum.HumanoidStateType.Swimming
			and state ~= Enum.HumanoidStateType.GettingUp then
			pcall(function() hum:SetStateEnabled(state, false) end)
		end
	end

	if not char:FindFirstChildOfClass("ForceField") then
		local ff = Instance.new("ForceField")
		ff.Visible = false
		ff.Parent = char
	end

	for _, v in ipairs(char:GetDescendants()) do
		if v:IsA("TouchTransmitter") or v:IsA("TouchInterest") then
			pcall(function() v:Destroy() end)
		end
	end

	for _, name in ipairs({"HumanoidRootPart", "Torso", "UpperTorso", "LowerTorso", "Head"}) do
		local p = char:FindFirstChild(name)
		if p and p:IsA("BasePart") then
			p.CanCollide = true
		end
	end

	if root then
		root:GetPropertyChangedSignal("AssemblyLinearVelocity"):Connect(function()
			if active and root and root.AssemblyLinearVelocity.Magnitude > 80 then
				root.AssemblyLinearVelocity = Vector3.zero
				root.AssemblyAngularVelocity = Vector3.zero
			end
		end)
		lastSafeCF = root.CFrame
	end

	pcall(function()
		local mt = getrawmetatable(hum)
		if mt and not metaHooked then
			setreadonly(mt, false)
			local old = mt.__newindex
			mt.__newindex = function(self, key, value)
				if key == "Health" and typeof(value) == "number" and value < 1e9 then
					value = 1e9
				elseif key == "MaxHealth" then
					value = 1e9
				end
				return old(self, key, value)
			end
			setreadonly(mt, true)
			metaHooked = true
		end
	end)

	hum.HealthChanged:Connect(function(hp)
		if active and hum and hum.Parent and hp < 1e9 then
			hum.Health = 1e9
			hum.MaxHealth = 1e9
			hum:SetStateEnabled(Enum.HumanoidStateType.Dead, false)
			pcall(function() hum:ChangeState(Enum.HumanoidStateType.Running) end)
		end
	end)
end

-------------------------------------------------
-- CURRENTANGLE V4
-------------------------------------------------
local function loadCurrentAngle()
	print("[*] Loading CurrentAngle V4...")
	local oldsettings = settings
	local s = _G
	s["Use default animations"] = true
	s["Local character transparency level"] = 1
	s["Disable character scripts"] = true
	s["Fake character should collide"] = true
	s["Parent real character to fake character"] = false
	s["Respawn character"] = true
	s["Instant respawn"] = false
	s["Hide HumanoidRootPart"] = false
	s["PermaDeath fake character"] = true
	s["R15 Reanimate"] = false
	s["Click Fling"] = false
	s["Anti-Fling"] = true
	s["Client sided display mode"] = 1
	s["Fallback prompt"] = false
	s["Respawn mode"] = "BreakJoints"
	s["Names to exclude from transparency"] = {}

	local ok = pcall(function()
		loadstring(game:HttpGet("https://raw.githubusercontent.com/somethingsimade/CurrentAngleV4/refs/heads/main/v4.lua"))()
	end)
	if not ok then
		ok = pcall(function()
			loadstring(game:HttpGet("https://raw.githubusercontent.com/somethingsimade/CurrentAngleV4/refs/heads/main/currentanglev2.5.lua"))()
		end)
	end
	settings = oldsettings
	if ok then print("[+] CurrentAngle loaded") end
	return ok
end

-------------------------------------------------
-- FE-REANIMATOR-V3
-------------------------------------------------
local function loadFEReanimatorV3()
	print("[*] Loading FE-Reanimator-v3...")
	local ok = pcall(function()
		loadstring(game:HttpGet("https://raw.githubusercontent.com/olidragon210/fe-reanimator-v3/main/reanimatorv3"))()
	end)
	if ok then print("[+] FE-Reanimator-v3 loaded") end
	return ok
end

-------------------------------------------------
-- FE HEADLESS (Nullware)
-------------------------------------------------
local function loadFEHeadless()
	print("[*] Loading FE Headless / Nullware...")
	_G.UnReanimateKey = "q"
	_G.ReanimateKey = "e"
	_G.R6ToggleKey = "r"
	_G.GodmodeToggleKey = "t"
	_G.CharacterBug = false
	_G.GodMode = true
	_G.R6 = false
	_G.FastLoading = true
	_G.AutoReanimate = true

	local ok = pcall(function()
		loadstring(game:HttpGet("https://paste.ee/r/e4oZ2/0"))()
	end)
	if ok then print("[+] FE Headless loaded") end
	return ok
end

-------------------------------------------------
-- REANIMATE LOADER
-------------------------------------------------
local function startReanimate()
	if reanimated then return end
	reanimated = true

	local success = loadCurrentAngle()
	if not success then
		success = loadFEReanimatorV3()
	end
	if not success then
		success = loadFEHeadless()
	end

	-- Re-apply Motor6Ds after reanimate
	task.delay(0.5, function()
		local char = player.Character
		if char then setupMotor6Ds(char) end
	end)

	if success then
		print("[+] Reanimate active")
	else
		warn("All reanimates failed")
		reanimated = false
	end
end

-------------------------------------------------
-- DRAGGABLE GUI
-------------------------------------------------
local function createGui()
	local gui = Instance.new("ScreenGui")
	gui.Name = "GodmodeToggle"
	gui.ResetOnSpawn = false
	gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
	gui.Parent = player:WaitForChild("PlayerGui")

	local frame = Instance.new("Frame")
	frame.Size = UDim2.new(0, 160, 0, 70)
	frame.Position = UDim2.new(0, 20, 0.4, 0)
	frame.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
	frame.BorderSizePixel = 0
	frame.Active = true
	frame.Draggable = true
	frame.Parent = gui

	Instance.new("UICorner", frame).CornerRadius = UDim.new(0, 8)
	local stroke = Instance.new("UIStroke", frame)
	stroke.Color = Color3.fromRGB(0, 170, 255)
	stroke.Thickness = 1.5

	local title = Instance.new("TextLabel", frame)
	title.Size = UDim2.new(1, 0, 0, 22)
	title.BackgroundTransparency = 1
	title.Text = "GODMODE"
	title.TextColor3 = Color3.fromRGB(0, 200, 255)
	title.Font = Enum.Font.GothamBold
	title.TextSize = 13

	local toggle = Instance.new("TextButton", frame)
	toggle.Size = UDim2.new(1, -16, 0, 32)
	toggle.Position = UDim2.new(0, 8, 0, 28)
	toggle.BackgroundColor3 = Color3.fromRGB(0, 140, 60)
	toggle.Text = "REANIMATE: OFF"
	toggle.TextColor3 = Color3.fromRGB(255, 255, 255)
	toggle.Font = Enum.Font.GothamBold
	toggle.TextSize = 12
	Instance.new("UICorner", toggle).CornerRadius = UDim.new(0, 6)

	toggle.MouseButton1Click:Connect(function()
		if reanimated then
			toggle.Text = "ALREADY ACTIVE"
			toggle.BackgroundColor3 = Color3.fromRGB(80, 80, 80)
			return
		end
		toggle.Text = "LOADING..."
		toggle.BackgroundColor3 = Color3.fromRGB(180, 120, 0)
		startReanimate()
		task.wait(0.8)
		if reanimated then
			toggle.Text = "REANIMATE: ON"
			toggle.BackgroundColor3 = Color3.fromRGB(0, 140, 60)
		else
			toggle.Text = "FAILED - RETRY"
			toggle.BackgroundColor3 = Color3.fromRGB(160, 40, 40)
			task.wait(1.5)
			toggle.Text = "REANIMATE: OFF"
			toggle.BackgroundColor3 = Color3.fromRGB(0, 140, 60)
		end
	end)
end

-------------------------------------------------
-- SETUP
-------------------------------------------------
local function onCharacter(char)
	task.wait(0.35)
	metaHooked = false
	lastSafeCF = nil
	maxGodmode(char)

	local hum = char:WaitForChild("Humanoid", 5)
	if not hum then return end

	hum.Died:Connect(function()
		task.wait(0.12)
		startReanimate()
	end)

	hum.HealthChanged:Connect(function(hp)
		if hp <= 0 and not reanimated then
			task.wait(0.08)
			startReanimate()
		end
	end)
end

player.CharacterAdded:Connect(onCharacter)
if player.Character then
	onCharacter(player.Character)
end

createGui()

RunService.Heartbeat:Connect(function()
	if not active then return end
	local char = player.Character
	if not char then return end

	local hum = char:FindFirstChildOfClass("Humanoid")
	if hum and hum.Health < 1e9 then
		hum.Health = 1e9
		hum.MaxHealth = 1e9
		hum:SetStateEnabled(Enum.HumanoidStateType.Dead, false)
	end

	antiFall(char)
end)

print("=== GODMODE + MOTOR6D + CURRENTANGLE + FE-REANIM + FE HEADLESS ===")
print("Motor6D joints locked (RootJoint, Neck, Shoulders, Hips)")
print("Drag GUI | Click to reanimate")
