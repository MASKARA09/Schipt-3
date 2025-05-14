local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Camera = workspace.CurrentCamera
local localPlayer = Players.LocalPlayer

local screenGui = Instance.new("ScreenGui", localPlayer:WaitForChild("PlayerGui"))
screenGui.Name = "ESP_UI"
screenGui.ResetOnSpawn = false

local espEnabled = true
local aimbotEnabled = false
local aimbotMaxDistance = 100 -- distância em studs
local fovRadius = 100 -- raio do círculo de FOV em pixels

-- Círculo FOV com Frame
local fovFrame = Instance.new("Frame", screenGui)
fovFrame.Size = UDim2.new(0, fovRadius * 2, 0, fovRadius * 2)
fovFrame.Position = UDim2.new(0.5, -fovRadius, 0.5, -fovRadius)
fovFrame.BackgroundColor3 = Color3.new(1, 1, 1)
fovFrame.BackgroundTransparency = 0.6
fovFrame.BorderSizePixel = 0

local fovCorner = Instance.new("UICorner", fovFrame)
fovCorner.CornerRadius = UDim.new(1, 0)

local espObjects = {}

-- Criar elementos visuais para um jogador
local function makeESPFor(player)
	local frame = Instance.new("Frame", screenGui)
	frame.BackgroundColor3 = Color3.new(1, 0, 0)
	frame.BorderSizePixel = 1
	frame.BackgroundTransparency = 0.7
	frame.Visible = false

	local text = Instance.new("TextLabel", screenGui)
	text.Size = UDim2.new(0, 200, 0, 20)
	text.TextColor3 = Color3.new(1, 1, 1)
	text.BackgroundTransparency = 1
	text.TextScaled = true
	text.Font = Enum.Font.SourceSansBold
	text.Visible = false

	return { Frame = frame, Text = text }
end

-- Atualizar a posição dos elementos ESP
local function createOrUpdateESP(player, boxTable)
	local char = player.Character
	if not char or not char:FindFirstChild("HumanoidRootPart") or not char:FindFirstChild("Head") then return end
	if player == localPlayer or player.Team == localPlayer.Team then
		boxTable.Frame.Visible = false
		boxTable.Text.Visible = false
		return
	end

	local root = char.HumanoidRootPart
	local head = char.Head
	local pos, visible = Camera:WorldToViewportPoint(root.Position)
	if not visible then
		boxTable.Frame.Visible = false
		boxTable.Text.Visible = false
		return
	end

	local distance = (localPlayer.Character.HumanoidRootPart.Position - root.Position).Magnitude
	local scale = 1 / (pos.Z + 1) * 100
	local width, height = 40 * scale, 60 * scale

	boxTable.Frame.Visible = true
	boxTable.Frame.Size = UDim2.new(0, width, 0, height)
	boxTable.Frame.Position = UDim2.new(0, pos.X - width/2, 0, pos.Y - height/2)

	boxTable.Text.Visible = true
	boxTable.Text.Text = string.format("%s [%.0f]", player.Name, distance)
	boxTable.Text.Position = UDim2.new(0, pos.X - width/2, 0, pos.Y - height/2 - 20)
end

-- Encontrar inimigo mais próximo dentro do FOV e alcance
local function getClosestEnemy()
	local closestDist = math.huge
	local target = nil
	local center = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)

	for _, player in pairs(Players:GetPlayers()) do
		if player ~= localPlayer and player.Team ~= localPlayer.Team then
			local char = player.Character
			if char and char:FindFirstChild("Head") then
				local head = char.Head
				local screenPos, onScreen = Camera:WorldToViewportPoint(head.Position)
				local distToPlayer = (localPlayer.Character.HumanoidRootPart.Position - head.Position).Magnitude
				local distToCenter = (Vector2.new(screenPos.X, screenPos.Y) - center).Magnitude

				if onScreen and distToPlayer <= aimbotMaxDistance and distToCenter <= fovRadius then
					if distToCenter < closestDist then
						closestDist = distToCenter
						target = head
					end
				end
			end
		end
	end

	return target
end

-- Atualização a cada frame
RunService.RenderStepped:Connect(function()
	-- Atualizar ESP
	for _, player in pairs(Players:GetPlayers()) do
		if not espObjects[player] then
			espObjects[player] = makeESPFor(player)
		end

		if espEnabled then
			createOrUpdateESP(player, espObjects[player])
		else
			espObjects[player].Frame.Visible = false
			espObjects[player].Text.Visible = false
		end
	end
end)

-- Aimbot ao pressionar E
UserInputService.InputBegan:Connect(function(input, gpe)
	if gpe then return end
	if input.KeyCode == Enum.KeyCode.E and aimbotEnabled then
		local targetHead = getClosestEnemy()
		if targetHead then
			Camera.CFrame = CFrame.new(Camera.CFrame.Position, targetHead.Position)
		end
	end
end)

-- Criar botão
local function createButton(text, pos, callback)
	local btn = Instance.new("TextButton", screenGui)
	btn.Size = UDim2.new(0, 140, 0, 35)
	btn.Position = pos
	btn.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
	btn.TextColor3 = Color3.new(1, 1, 1)
	btn.Font = Enum.Font.SourceSansBold
	btn.TextSize = 18
	btn.Text = text
	btn.BorderSizePixel = 0
	btn.MouseButton1Click:Connect(callback)
	return btn
end

-- Botões de controle
local espBtn = createButton("ESP: ON", UDim2.new(0, 20, 0, 20), function()
	espEnabled = not espEnabled
	espBtn.Text = espEnabled and "ESP: ON" or "ESP: OFF"
	espBtn.BackgroundColor3 = espEnabled and Color3.fromRGB(0, 170, 0) or Color3.fromRGB(50, 50, 50)
end)

local aimbotBtn = createButton("Aimbot: OFF", UDim2.new(0, 20, 0, 60), function()
	aimbotEnabled = not aimbotEnabled
	aimbotBtn.Text = aimbotEnabled and "Aimbot: ON" or "Aimbot: OFF"
	aimbotBtn.BackgroundColor3 = aimbotEnabled and Color3.fromRGB(0, 170, 0) or Color3.fromRGB(50, 50, 50)
end)
