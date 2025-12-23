local Players = game:GetService("Players")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")

local player = Players.LocalPlayer

-- GUI
local gui = Instance.new("ScreenGui")
gui.Name = "FixedHub"
gui.ResetOnSpawn = false
if player.PlayerGui then
    gui.Parent = player.PlayerGui
else
    player:WaitForChild("PlayerGui")
    gui.Parent = player.PlayerGui
end

-- Função Desync V3
local function activateDesync()
    local FFlags = {
        GameNetPVHeaderRotationalVelocityZeroCutoffExponent = -5000,
        LargeReplicatorWrite5 = true,
        LargeReplicatorEnabled9 = true,
        AngularVelociryLimit = 360,
        TimestepArbiterVelocityCriteriaThresholdTwoDt = 2147483646,
        S2PhysicsSenderRate = 15000,
        DisableDPIScale = true,
        MaxDataPacketPerSend = 2147483647,
        PhysicsSenderMaxBandwidthBps = 20000,
        TimestepArbiterHumanoidLinearVelThreshold = 21,
        MaxMissedWorldStepsRemembered = -2147483648,
        PlayerHumanoidPropertyUpdateRestrict = true,
        SimDefaultHumanoidTimestepMultiplier = 0,
        StreamJobNOUVolumeLengthCap = 2147483647,
        DebugSendDistInSteps = -2147483648,
        GameNetDontSendRedundantNumTimes = 1,
        CheckPVLinearVelocityIntegrateVsDeltaPositionThresholdPercent = 1,
        CheckPVDifferencesForInterpolationMinVelThresholdStudsPerSecHundredth = 1,
        LargeReplicatorSerializeRead3 = true,
        ReplicationFocusNouExtentsSizeCutoffForPauseStuds = 2147483647,
        CheckPVCachedVelThresholdPercent = 10,
        CheckPVDifferencesForInterpolationMinRotVelThresholdRadsPerSecHundredth = 1,
        GameNetDontSendRedundantDeltaPositionMillionth = 1,
        InterpolationFrameVelocityThresholdMillionth = 5,
        StreamJobNOUVolumeCap = 2147483647,
        InterpolationFrameRotVelocityThresholdMillionth = 5,
        CheckPVCachedRotVelThresholdPercent = 10,
        WorldStepMax = 30,
        InterpolationFramePositionThresholdMillionth = 5,
        TimestepArbiterHumanoidTurningVelThreshold = 1,
        SimOwnedNOUCountThresholdMillionth = 2147483647,
        GameNetPVHeaderLinearVelocityZeroCutoffExponent = -5000,
        NextGenReplicatorEnabledWrite4 = true,
        TimestepArbiterOmegaThou = 1073741823,
        MaxAcceptableUpdateDelay = 1,
        LargeReplicatorSerializeWrite4 = true
    }

    local function respawnar(plr)
        local rcdEnabled = false
        
        local success, result = pcall(function()
            return game:GetService("Workspace").RejectCharacterDeletions ~= Enum.RejectCharacterDeletions.Disabled
        end)
        
        if success then
            rcdEnabled = result
        end
        
        if rcdEnabled then
            task.wait(Players.RespawnTime - 0.1)
            if plr.Character then
                plr.Character:BreakJoints()
            end
        else
            local char = plr.Character
            if char then
                local hum = char:FindFirstChildWhichIsA('Humanoid')
                if hum then
                    hum:ChangeState(Enum.HumanoidStateType.Dead)
                end
                char:ClearAllChildren()
                local newChar = Instance.new('Model')
                newChar.Parent = workspace
                plr.Character = newChar
                task.wait()
                plr.Character = char
                newChar:Destroy()
            end
        end
    end

    for name, value in pairs(FFlags) do
        pcall(function()
            setfflag(tostring(name), tostring(value))
        end)
    end
    
    respawnar(player)
    
    return true
end

-- Função Steal Speed (velocidade 29)
local function stealSpeed()
    local character = player.Character
    if not character then return false end
    
    local humanoid = character:FindFirstChildOfClass("Humanoid")
    if not humanoid then return false end
    
    -- Alternar entre velocidade normal e 29
    if humanoid.WalkSpeed == 29 then
        humanoid.WalkSpeed = 16
        return false
    else
        humanoid.WalkSpeed = 29
        return true
    end
end

-- Função Steal Floor
local function stealFloor()
    local character = player.Character
    if not character then return false end
    
    local humanoidRootPart = character:FindFirstChild("HumanoidRootPart")
    if not humanoidRootPart then return false end
    
    -- Procurar pelo chão mais próximo
    local rayOrigin = humanoidRootPart.Position
    local rayDirection = Vector3.new(0, -50, 0)
    local ray = Ray.new(rayOrigin, rayDirection)
    
    local hitPart, hitPosition = workspace:FindPartOnRay(ray, character)
    
    if hitPart then
        -- Criar uma cópia do chão
        local floorCopy = hitPart:Clone()
        floorCopy.Parent = workspace
        floorCopy.Position = humanoidRootPart.Position + Vector3.new(0, -5, 0)
        floorCopy.Anchored = true
        floorCopy.CanCollide = true
        
        -- Efeito visual
        floorCopy.Transparency = 0.7
        floorCopy.Color = Color3.fromRGB(255, 100, 100)
        floorCopy.Material = Enum.Material.Neon
        
        -- Destruir após 10 segundos
        task.delay(10, function()
            if floorCopy and floorCopy.Parent then
                floorCopy:Destroy()
            end
        end)
        
        return true
    end
    
    return false
end

-- MAIN FRAME
local main = Instance.new("Frame")
main.Size = UDim2.fromOffset(280, 300)
main.Position = UDim2.fromScale(0.05, 0.25)
main.BackgroundColor3 = Color3.fromRGB(15, 15, 15)
main.BackgroundTransparency = 0.1
main.BorderSizePixel = 0
main.Active = true
main.ClipsDescendants = true
main.Parent = gui

-- CANTOS
local corner = Instance.new("UICorner")
corner.CornerRadius = UDim.new(0, 16)
corner.Parent = main

-- BORDA COM GRADIENTE
local stroke = Instance.new("UIStroke")
stroke.Color = Color3.fromRGB(180, 0, 0)
stroke.Thickness = 2
stroke.Parent = main

-- BORDA ANIMADA
task.spawn(function()
	while gui.Parent do
		TweenService:Create(stroke, TweenInfo.new(1.5), {Color = Color3.fromRGB(220, 40, 40)}):Play()
		task.wait(1.5)
		TweenService:Create(stroke, TweenInfo.new(1.5), {Color = Color3.fromRGB(180, 0, 0)}):Play()
		task.wait(1.5)
	end
end)

-- CABEÇALHO
local header = Instance.new("Frame")
header.Size = UDim2.new(1, 0, 0, 50)
header.Position = UDim2.fromOffset(0, 0)
header.BackgroundColor3 = Color3.fromRGB(180, 0, 0)
header.BackgroundTransparency = 0.2
header.BorderSizePixel = 0
header.Parent = main

local headerCorner = Instance.new("UICorner")
headerCorner.CornerRadius = UDim.new(0, 16)
headerCorner.Name = "HeaderCorner"
headerCorner.Parent = header

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, 0, 1, 0)
title.BackgroundTransparency = 1
title.Text = "Apple Hub Stealer"
title.TextColor3 = Color3.fromRGB(255, 255, 255)
title.Font = Enum.Font.GothamBlack
title.TextSize = 20
title.TextStrokeTransparency = 0.7
title.Parent = header

local subtitle = Instance.new("TextLabel")
subtitle.Size = UDim2.new(1, 0, 0, 20)
subtitle.Position = UDim2.fromOffset(0, 28)
subtitle.BackgroundTransparency = 1
subtitle.Text = "v1.6.7 | INSERT TO HIDE"
subtitle.TextColor3 = Color3.fromRGB(200, 200, 200)
subtitle.Font = Enum.Font.Gotham
subtitle.TextSize = 11
subtitle.Parent = header

-- SISTEMA DE ARRASTAR
do
	local dragging = false
	local dragStart
	local startPos
	
	header.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 then
			dragging = true
			dragStart = input.Position
			startPos = main.Position
			
			TweenService:Create(header, TweenInfo.new(0.1), {
				BackgroundTransparency = 0.1
			}):Play()
		end
	end)
	
	UserInputService.InputChanged:Connect(function(input)
		if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
			local delta = input.Position - dragStart
			main.Position = UDim2.new(
				startPos.X.Scale, startPos.X.Offset + delta.X,
				startPos.Y.Scale, startPos.Y.Offset + delta.Y
			)
		end
	end)
	
	UserInputService.InputEnded:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 then
			dragging = false
			TweenService:Create(header, TweenInfo.new(0.2), {
				BackgroundTransparency = 0.2
			}):Play()
		end
	end)
end

-- CONTAINER COM SCROLL
local container = Instance.new("ScrollingFrame")
container.Name = "Container"
container.Size = UDim2.new(1, -20, 1, -80)
container.Position = UDim2.fromOffset(10, 70)
container.BackgroundTransparency = 1
container.BorderSizePixel = 0
container.ScrollBarThickness = 3
container.ScrollBarImageColor3 = Color3.fromRGB(180, 0, 0)
container.ScrollBarImageTransparency = 0.6
container.CanvasSize = UDim2.new(0, 0, 0, 0)
container.AutomaticCanvasSize = Enum.AutomaticSize.Y
container.Parent = main

-- LAYOUT
local layout = Instance.new("UIListLayout")
layout.Padding = UDim.new(0, 12)
layout.HorizontalAlignment = Enum.HorizontalAlignment.Center
layout.Parent = container

-- FUNÇÃO PARA CRIAR SEÇÃO
local function createSection(titleText)
	local sectionFrame = Instance.new("Frame")
	sectionFrame.Size = UDim2.new(1, 0, 0, 36)
	sectionFrame.BackgroundTransparency = 1
	
	local titleLabel = Instance.new("TextLabel")
	titleLabel.Size = UDim2.new(1, 0, 1, 0)
	titleLabel.BackgroundTransparency = 1
	titleLabel.Text = "━━ " .. titleText .. " ━━"
	titleLabel.TextColor3 = Color3.fromRGB(180, 0, 0)
	titleLabel.Font = Enum.Font.GothamBold
	titleLabel.TextSize = 16
	titleLabel.TextStrokeTransparency = 0.8
	titleLabel.Parent = sectionFrame
	
	sectionFrame.Parent = container
	return sectionFrame
end

-- ARMAZENAR BOTÕES PARA CONTROLE
local buttons = {}

-- FUNÇÃO PARA CRIAR BOTÃO
local function createButton(text, func, colorType)
	local buttonFrame = Instance.new("Frame")
	buttonFrame.Size = UDim2.new(1, 0, 0, 46)
	buttonFrame.BackgroundTransparency = 1
	
	local button = Instance.new("TextButton")
	button.Size = UDim2.new(1, -10, 1, 0)
	button.Position = UDim2.fromOffset(5, 0)
	button.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
	button.BackgroundTransparency = 0.1
	button.Text = ""
	button.BorderSizePixel = 0
	button.AutoButtonColor = false
	button.Parent = buttonFrame
	
	local corner = Instance.new("UICorner")
	corner.CornerRadius = UDim.new(0, 10)
	corner.Parent = button
	
	local stroke = Instance.new("UIStroke")
	stroke.Color = Color3.fromRGB(60, 60, 60)
	stroke.Thickness = 1.5
	stroke.Parent = button
	
	-- Texto do botão
	local buttonText = Instance.new("TextLabel")
	buttonText.Size = UDim2.new(1, -50, 1, 0)
	buttonText.Position = UDim2.fromOffset(15, 0)
	buttonText.BackgroundTransparency = 1
	buttonText.Text = text
	buttonText.TextColor3 = Color3.fromRGB(220, 220, 220)
	buttonText.Font = Enum.Font.GothamSemibold
	buttonText.TextSize = 14
	buttonText.TextXAlignment = Enum.TextXAlignment.Left
	buttonText.Parent = button
	
	-- Indicador de status (toggle)
	local toggleCircle = Instance.new("Frame")
	toggleCircle.Size = UDim2.new(0, 20, 0, 20)
	toggleCircle.Position = UDim2.new(1, -35, 0.5, -10)
	toggleCircle.AnchorPoint = Vector2.new(1, 0.5)
	toggleCircle.BackgroundColor3 = Color3.fromRGB(80, 80, 80)
	toggleCircle.BorderSizePixel = 0
	toggleCircle.Parent = button
	
	local toggleCorner = Instance.new("UICorner")
	toggleCorner.CornerRadius = UDim.new(1, 0)
	toggleCorner.Parent = toggleCircle
	
	-- Armazenar referência
	buttons[text] = {
		frame = button,
		text = buttonText,
		stroke = stroke,
		toggle = toggleCircle,
		isActive = false,
		func = func,
		colorType = colorType or "green"
	}
	
	-- Efeitos e funcionalidade
	button.MouseEnter:Connect(function()
		if not buttons[text].isActive then
			TweenService:Create(button, TweenInfo.new(0.2), {
				BackgroundColor3 = Color3.fromRGB(40, 40, 40)
			}):Play()
			TweenService:Create(stroke, TweenInfo.new(0.2), {
				Color = Color3.fromRGB(100, 100, 100)
			}):Play()
		end
	end)
	
	button.MouseLeave:Connect(function()
		if not buttons[text].isActive then
			TweenService:Create(button, TweenInfo.new(0.2), {
				BackgroundColor3 = Color3.fromRGB(30, 30, 30)
			}):Play()
			TweenService:Create(stroke, TweenInfo.new(0.2), {
				Color = Color3.fromRGB(60, 60, 60)
			}):Play()
		end
	end)
	
	button.MouseButton1Click:Connect(function()
		local btnData = buttons[text]
		local wasActive = btnData.isActive
		
		-- Executar função
		local result = func()
		
		-- Atualizar estado
		if result ~= nil then
			btnData.isActive = result
		else
			btnData.isActive = not btnData.isActive
		end
		
		-- Aplicar efeitos visuais
		if btnData.isActive then
			-- Ativado
			local activeColor, activeStrokeColor, activeToggleColor
			
			if btnData.colorType == "blue" then
				activeColor = Color3.fromRGB(30, 60, 150)
				activeStrokeColor = Color3.fromRGB(0, 120, 255)
				activeToggleColor = Color3.fromRGB(0, 120, 255)
			elseif btnData.colorType == "red" then
				activeColor = Color3.fromRGB(80, 30, 30)
				activeStrokeColor = Color3.fromRGB(255, 50, 50)
				activeToggleColor = Color3.fromRGB(255, 50, 50)
			else -- verde padrão
				activeColor = Color3.fromRGB(30, 60, 30)
				activeStrokeColor = Color3.fromRGB(0, 180, 0)
				activeToggleColor = Color3.fromRGB(0, 220, 0)
			end
			
			TweenService:Create(button, TweenInfo.new(0.2), {
				BackgroundColor3 = activeColor
			}):Play()
			TweenService:Create(stroke, TweenInfo.new(0.2), {
				Color = activeStrokeColor,
				Thickness = 2
			}):Play()
			TweenService:Create(toggleCircle, TweenInfo.new(0.2), {
				BackgroundColor3 = activeToggleColor,
				Size = UDim2.new(0, 24, 0, 24)
			}):Play()
			TweenService:Create(buttonText, TweenInfo.new(0.2), {
				TextColor3 = Color3.fromRGB(255, 255, 255)
			}):Play()
			
		else
			-- Desativado
			TweenService:Create(button, TweenInfo.new(0.2), {
				BackgroundColor3 = Color3.fromRGB(30, 30, 30)
			}):Play()
			TweenService:Create(stroke, TweenInfo.new(0.2), {
				Color = Color3.fromRGB(60, 60, 60),
				Thickness = 1.5
			}):Play()
			TweenService:Create(toggleCircle, TweenInfo.new(0.2), {
				BackgroundColor3 = Color3.fromRGB(80, 80, 80),
				Size = UDim2.new(0, 20, 0, 20)
			}):Play()
			TweenService:Create(buttonText, TweenInfo.new(0.2), {
				TextColor3 = Color3.fromRGB(220, 220, 220)
			}):Play()
		end
	end)
	
	buttonFrame.Parent = container
	return button
end

-- CRIAR APENAS OS 3 BOTÕES PEDIDOS
task.wait(0.1)

-- CATEGORIA: HACKS
createSection("Functions")
createButton("DESYNC V3", activateDesync, "green")
createButton("STEAL SPEED", stealSpeed, "blue")
createButton("STEAL FLOOR, PAtCHED", stealFloor, "red")

-- Ajustar canvas size
layout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
	container.CanvasSize = UDim2.new(0, 0, 0, layout.AbsoluteContentSize.Y + 10)
end)

RunService.RenderStepped:Wait()
container.CanvasSize = UDim2.new(0, 0, 0, layout.AbsoluteContentSize.Y + 10)

-- Efeito de entrada
main.Position = UDim2.fromScale(0.05, 0.15)
main.BackgroundTransparency = 0.8
main.Size = UDim2.fromOffset(280, 0)

TweenService:Create(main, TweenInfo.new(0.7, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
	Position = UDim2.fromScale(0.05, 0.25),
	BackgroundTransparency = 0.1,
	Size = UDim2.fromOffset(280, 300)
}):Play()

-- Controle com Insert
UserInputService.InputBegan:Connect(function(input, gameProcessed)
	if not gameProcessed and input.KeyCode == Enum.KeyCode.Insert then
		gui.Enabled = not gui.Enabled
	end
end)
