-- [Mesma estrutura inicial do script anterior até a parte "Render Loop"]
-- Mantém as funções SetupGUI, IsVisible, GetClosestPlayerInFOV, etc.

-- Atualiza a estrutura de DrawingESP para incluir barra de vida
local function CreateHealth()
	local bar = Drawing.new("Line")
	bar.Thickness = 3
	bar.Visible = false
	return bar
end

for _, player in ipairs(Players:GetPlayers()) do
	if player ~= LocalPlayer then
		DrawingESP[player] = {
			Box = CreateBox(),
			Name = CreateName(),
			Health = CreateHealth(),
		}
	end
end

Players.PlayerAdded:Connect(function(player)
	if player ~= LocalPlayer then
		DrawingESP[player] = {
			Box = CreateBox(),
			Name = CreateName(),
			Health = CreateHealth(),
		}
	end
end)

Players.PlayerRemoving:Connect(function(player)
	if DrawingESP[player] then
		for _, v in pairs(DrawingESP[player]) do
			v:Remove()
		end
		DrawingESP[player] = nil
	end
end)

-- ATUALIZADO: Só considera inimigos (diferente TeamColor)
local function IsEnemy(player)
	if player.Team and LocalPlayer.Team then
		return player.TeamColor ~= LocalPlayer.TeamColor
	end
	return true -- se não tem time, considera inimigo
end

-- Atualizado: só considera inimigos no aimbot
local function GetClosestEnemyInFOV()
	local closest = nil
	local shortest = math.huge

	for _, player in ipairs(Players:GetPlayers()) do
		if player ~= LocalPlayer and IsEnemy(player) and player.Character and player.Character:FindFirstChild("Head") then
			local head = player.Character.Head
			local distance = (head.Position - LocalPlayer.Character.HumanoidRootPart.Position).Magnitude

			if distance <= maxDistance and IsVisible(head) then
				local screenPos, onScreen = Camera:WorldToViewportPoint(head.Position)
				if onScreen then
					local mouseDist = (Vector2.new(screenPos.X, screenPos.Y) - Vector2.new(Mouse.X, Mouse.Y)).Magnitude
					if mouseDist < shortest then
						shortest = mouseDist
						closest = head
					end
				end
			end
		end
	end
	return closest
end

-- Atualizado loop de renderização
RunService.RenderStepped:Connect(function()
	-- Aimbot
	if AimbotEnabled then
		local target = GetClosestEnemyInFOV()
		if target then
			local look = (target.Position - Camera.CFrame.Position).Unit
			Camera.CFrame = CFrame.new(Camera.CFrame.Position, Camera.CFrame.Position + look)
		end
	end

	-- ESP
	for _, player in ipairs(Players:GetPlayers()) do
		if player ~= LocalPlayer and IsEnemy(player) and player.Character and player.Character:FindFirstChild("HumanoidRootPart") and player.Character:FindFirstChild("Head") and player.Character:FindFirstChild("Humanoid") then
			local hrp = player.Character.HumanoidRootPart
			local head = player.Character.Head
			local humanoid = player.Character.Humanoid
			local box = DrawingESP[player].Box
			local name = DrawingESP[player].Name
			local healthBar = DrawingESP[player].Health

			local rootPos, onScreen = Camera:WorldToViewportPoint(hrp.Position)
			local headPos = Camera:WorldToViewportPoint(head.Position + Vector3.new(0, 0.5, 0))
			local footPos = Camera:WorldToViewportPoint(hrp.Position - Vector3.new(0, 3, 0))

			local height = math.abs(footPos.Y - headPos.Y)
			local width = height / 2

			if onScreen and ESPEnabled then
				box.Size = Vector2.new(width, height)
				box.Position = Vector2.new(rootPos.X - width / 2, rootPos.Y - height / 2)
				box.Visible = true
				box.Color = IsVisible(head) and Color3.fromRGB(0, 255, 0) or Color3.fromRGB(255, 0, 0)

				name.Text = player.Name
				name.Position = Vector2.new(rootPos.X, rootPos.Y - height / 2 - 15)
				name.Color = box.Color
				name.Visible = true

				-- Health bar (esquerda da box)
				local hpPercent = math.clamp(humanoid.Health / humanoid.MaxHealth, 0, 1)
				local top = Vector2.new(box.Position.X - 6, box.Position.Y)
				local bottom = Vector2.new(top.X, box.Position.Y + height)
				local mid = Vector2.new(top.X, top.Y + height * (1 - hpPercent))

				healthBar.From = bottom
				healthBar.To = mid
				healthBar.Color = Color3.fromRGB(0, 255, 0)
				healthBar.Visible = true
			else
				box.Visible = false
				name.Visible = false
				healthBar.Visible = false
			end
		elseif DrawingESP[player] then
			DrawingESP[player].Box.Visible = false
			DrawingESP[player].Name.Visible = false
			DrawingESP[player].Health.Visible = false
		end
	end
end)
