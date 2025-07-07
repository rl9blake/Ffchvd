local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
local Camera = Workspace.CurrentCamera

local LocalPlayer = Players.LocalPlayer
local Mouse = LocalPlayer:GetMouse()

-- GUI
local ScreenGui = Instance.new("ScreenGui", LocalPlayer:WaitForChild("PlayerGui"))
ScreenGui.Name = "AimbotESP_GUI"

local AimbotButton = Instance.new("TextButton", ScreenGui)
AimbotButton.Size = UDim2.new(0, 120, 0, 40)
AimbotButton.Position = UDim2.new(0, 10, 0, 10)
AimbotButton.Text = "Aimbot: OFF"
AimbotButton.BackgroundColor3 = Color3.fromRGB(255, 0, 0)
AimbotButton.TextScaled = true

local ESPButton = Instance.new("TextButton", ScreenGui)
ESPButton.Size = UDim2.new(0, 120, 0, 40)
ESPButton.Position = UDim2.new(0, 10, 0, 60)
ESPButton.Text = "ESP: OFF"
ESPButton.BackgroundColor3 = Color3.fromRGB(255, 0, 0)
ESPButton.TextScaled = true

-- Estado
local AimbotEnabled = false
local ESPEnabled = false

AimbotButton.MouseButton1Click:Connect(function()
	AimbotEnabled = not AimbotEnabled
	AimbotButton.Text = AimbotEnabled and "Aimbot: ON" or "Aimbot: OFF"
	AimbotButton.BackgroundColor3 = AimbotEnabled and Color3.fromRGB(0, 255, 0) or Color3.fromRGB(255, 0, 0)
end)

ESPButton.MouseButton1Click:Connect(function()
	ESPEnabled = not ESPEnabled
	ESPButton.Text = ESPEnabled and "ESP: ON" or "ESP: OFF"
	ESPButton.BackgroundColor3 = ESPEnabled and Color3.fromRGB(0, 255, 0) or Color3.fromRGB(255, 0, 0)
end)

-- Raycast para visibilidade
local rayParams = RaycastParams.new()
rayParams.FilterType = Enum.RaycastFilterType.Blacklist
rayParams.IgnoreWater = true

local function IsVisible(part)
	local origin = Camera.CFrame.Position
	local direction = (part.Position - origin)
	rayParams.FilterDescendantsInstances = {LocalPlayer.Character}
	local result = Workspace:Raycast(origin, direction, rayParams)
	return not result or result.Instance:IsDescendantOf(part.Parent)
end

-- Aimbot com verificação de distância (20 studs)
local function GetClosestVisiblePlayer(maxDistance)
	local closest = nil
	local shortest = math.huge

	for _, player in ipairs(Players:GetPlayers()) do
		if player ~= LocalPlayer and player.Character and player.Character:FindFirstChild("Head") then
			local head = player.Character.Head
			local screenPos, onScreen = Camera:WorldToViewportPoint(head.Position)
			local distanceToPlayer = (head.Position - LocalPlayer.Character.HumanoidRootPart.Position).Magnitude

			if onScreen and IsVisible(head) and distanceToPlayer <= maxDistance then
				local mouseDistance = (Vector2.new(screenPos.X, screenPos.Y) - Vector2.new(Mouse.X, Mouse.Y)).Magnitude
				if mouseDistance < shortest then
					shortest = mouseDistance
					closest = player
				end
			end
		end
	end
	return closest
end

-- ESP (Box + Nome)
local DrawingESP = {}

local function CreateBox()
	local box = Drawing.new("Square")
	box.Thickness = 1.5
	box.Filled = false
	box.Visible = false
	return box
end

local function CreateName()
	local text = Drawing.new("Text")
	text.Size = 14
	text.Center = true
	text.Outline = true
	text.Visible = false
	return text
end

for _, player in ipairs(Players:GetPlayers()) do
	if player ~= LocalPlayer then
		DrawingESP[player] = {
			Box = CreateBox(),
			Name = CreateName(),
		}
	end
end

Players.PlayerAdded:Connect(function(player)
	if player ~= LocalPlayer then
		DrawingESP[player] = {
			Box = CreateBox(),
			Name = CreateName(),
		}
	end
end)

Players.PlayerRemoving:Connect(function(player)
	if DrawingESP[player] then
		for _, obj in pairs(DrawingESP[player]) do
			obj:Remove()
		end
		DrawingESP[player] = nil
	end
end)

-- Loop
RunService.RenderStepped:Connect(function()
	-- Aimbot
	if AimbotEnabled then
		local maxDistance = 20 -- Alcance do Aimbot
		local target = GetClosestVisiblePlayer(maxDistance)
		if target and target.Character and target.Character:FindFirstChild("Head") then
			local headPos = target.Character.Head.Position
			local look = (headPos - Camera.CFrame.Position).Unit
			Camera.CFrame = CFrame.new(Camera.CFrame.Position, Camera.CFrame.Position + look)
		end
	end

	-- ESP
	for _, player in ipairs(Players:GetPlayers()) do
		if player ~= LocalPlayer and player.Character and player.Character:FindFirstChild("HumanoidRootPart") and player.Character:FindFirstChild("Head") then
			local hrp = player.Character.HumanoidRootPart
			local head = player.Character.Head
			local box = DrawingESP[player].Box
			local name = DrawingESP[player].Name

			local rootPos, onScreen = Camera:WorldToViewportPoint(hrp.Position)
			local headPos = Camera:WorldToViewportPoint(head.Position + Vector3.new(0, 0.5, 0))
			local footPos = Camera:WorldToViewportPoint(hrp.Position - Vector3.new(0, 3, 0))

			local height = math.abs(footPos.Y - headPos.Y)
			local width = height / 2

			if onScreen and ESPEnabled then
				box.Size = Vector2.new(width, height)
				box.Position = Vector2.new(rootPos.X - width / 2, rootPos.Y - height / 2)
				box.Visible = true

				local visible = IsVisible(head)
				box.Color = visible and Color3.fromRGB(0, 255, 0) or Color3.fromRGB(255, 0, 0)

				name.Text = player.Name
				name.Position = Vector2.new(rootPos.X, rootPos.Y - height / 2 - 15)
				name.Color = box.Color
				name.Visible = true
			else
				box.Visible = false
				name.Visible = false
			end
		elseif DrawingESP[player] then
			DrawingESP[player].Box.Visible = false
			DrawingESP[player].Name.Visible = false
		end
	end
end)
