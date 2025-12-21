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

-- Variável para controlar o teleporte teleguiado
local isTeleportingToBase = false
local teleportConnection = nil

-- Função para encontrar o spawn point original
local function findSpawnPoint()
    -- Procura pelo SpawnLocation padrão
    local spawnLocation = workspace:FindFirstChildOfClass("SpawnLocation")
    if spawnLocation then
        return spawnLocation.Position
    end
    
    -- Procura por qualquer coisa com "spawn" no nome
    for _, obj in pairs(workspace:GetDescendants()) do
        if obj:IsA("Part") and obj.Name:lower():find("spawn") then
            return obj.Position
        end
    end
    
    -- Se não encontrar, volta para a posição inicial do jogador
    if player.Character then
        local humanoidRootPart = player.Character:FindFirstChild("HumanoidRootPart")
        if humanoidRootPart then
            return humanoidRootPart.Position
        end
    end
    
    return Vector3.new(0, 5, 0)
end

-- Função para verificar linha de visão direta
local function hasDirectLineOfSight(startPos, endPos)
    local direction = (endPos - startPos)
    local distance = direction.Magnitude
    local ray = Ray.new(startPos + Vector3.new(0, 2, 0), direction)
    local hit, hitPosition = workspace:FindPartOnRayWithIgnoreList(ray, {player.Character})
    
    return not hit, hitPosition
end

-- Função para encontrar ponto de desvio
local function findBypassPoint(currentPos, targetPos, obstaclePos)
    -- Calcula vetores
    local toTarget = (targetPos - currentPos)
    local obstacleNormal = (obstaclePos - currentPos).Unit
    
    -- Tenta contornar pela direita
    local rightDir = obstacleNormal:Cross(Vector3.new(0, 1, 0))
    local rightPoint = currentPos + (rightDir * 5) + (toTarget.Unit * 5)
    
    -- Verifica se o ponto da direita tem linha de visão
    local clearRight, _ = hasDirectLineOfSight(currentPos, rightPoint)
    if clearRight then
        return rightPoint
    end
    
    -- Tenta contornar pela esquerda
    local leftDir = -rightDir
    local leftPoint = currentPos + (leftDir * 5) + (toTarget.Unit * 5)
    
    local clearLeft, _ = hasDirectLineOfSight(currentPos, leftPoint)
    if clearLeft then
        return leftPoint
    end
    
    -- Se não conseguir, tenta subir
    local upPoint = currentPos + Vector3.new(0, 10, 0) + (toTarget.Unit * 5)
    return upPoint
end

-- Função TP TO BASE Teleguiado DIRETO
local function teleportToBaseTeleguided()
    if isTeleportingToBase then
        return false
    end
    
    local character = player.Character
    if not character then
        return false
    end
    
    local humanoid = character:FindFirstChildOfClass("Humanoid")
    local humanoidRootPart = character:FindFirstChild("HumanoidRootPart")
    
    if not humanoid or not humanoidRootPart then
        return false
    end
    
    -- Encontrar ponto de spawn
    local targetPosition = findSpawnPoint()
    local startPosition = humanoidRootPart.Position
    
    -- Ajustar altura para a mesma do jogador
    targetPosition = Vector3.new(targetPosition.X, startPosition.Y, targetPosition.Z)
    
    -- Ativar estado de teleporte
    isTeleportingToBase = true
    
    -- Salvar estado original
    local originalWalkSpeed = humanoid.WalkSpeed
    local originalJumpPower = humanoid.JumpPower
    
    -- Configurar para movimento
    humanoid.WalkSpeed = 0
    humanoid.JumpPower = 0
    
    -- Criar BodyVelocity para movimento
    local bodyVelocity = Instance.new("BodyVelocity")
    bodyVelocity.Velocity = Vector3.new(0, 0, 0)
    bodyVelocity.MaxForce = Vector3.new(40000, 0, 40000)
    bodyVelocity.P = 1000
    bodyVelocity.Parent = humanoidRootPart
    
    -- Criar BodyGyro para manter orientação
    local bodyGyro = Instance.new("BodyGyro")
    bodyGyro.MaxTorque = Vector3.new(40000, 40000, 40000)
    bodyGyro.P = 1000
    bodyGyro.D = 100
    bodyGyro.CFrame = humanoidRootPart.CFrame
    bodyGyro.Parent = humanoidRootPart
    
    -- Criar efeito visual de guia
    local guideEffect = Instance.new("Part")
    guideEffect.Name = "TeleportGuide"
    guideEffect.Size = Vector3.new(1, 1, 1)
    guideEffect.Position = targetPosition
    guideEffect.Transparency = 0.3
    guideEffect.Color = Color3.fromRGB(0, 255, 0)
    guideEffect.Material = Enum.Material.Neon
    guideEffect.Anchored = true
    guideEffect.CanCollide = false
    guideEffect.Parent = workspace
    
    -- Sistema de movimento teleguiado DIRETO
    teleportConnection = RunService.Heartbeat:Connect(function(deltaTime)
        if not isTeleportingToBase or not humanoidRootPart or not humanoidRootPart.Parent then
            return
        end
        
        local currentPos = humanoidRootPart.Position
        local toTarget = targetPosition - currentPos
        local distanceToTarget = toTarget.Magnitude
        
        -- Se estiver muito perto, finalizar
        if distanceToTarget < 3 then
            isTeleportingToBase = false
            
            -- Posicionar exatamente no alvo
            humanoidRootPart.CFrame = CFrame.new(targetPosition)
            
            -- Limpar
            bodyVelocity:Destroy()
            bodyGyro:Destroy()
            guideEffect:Destroy()
            
            -- Restaurar
            humanoid.WalkSpeed = originalWalkSpeed
            humanoid.JumpPower = originalJumpPower
            
            if teleportConnection then
                teleportConnection:Disconnect()
                teleportConnection = nil
            end
            
            return
        end
        
        -- Verificar linha de visão direta
        local hasLOS, obstaclePos = hasDirectLineOfSight(currentPos, targetPosition)
        
        if hasLOS then
            -- Mover diretamente para o alvo
            local moveDirection = toTarget.Unit
            bodyVelocity.Velocity = moveDirection * 11
            bodyGyro.CFrame = CFrame.new(currentPos, currentPos + moveDirection)
        else
            -- Encontrar ponto para contornar
            local bypassPoint = findBypassPoint(currentPos, targetPosition, obstaclePos)
            local toBypass = bypassPoint - currentPos
            local bypassDistance = toBypass.Magnitude
            
            if bypassDistance < 2 then
                -- Já chegou no ponto de desvio, voltar a tentar linha direta
                bodyVelocity.Velocity = toTarget.Unit * 11
                bodyGyro.CFrame = CFrame.new(currentPos, currentPos + toTarget.Unit)
            else
                -- Mover para o ponto de desvio
                local bypassDirection = toBypass.Unit
                bodyVelocity.Velocity = bypassDirection * 11
                bodyGyro.CFrame = CFrame.new(currentPos, currentPos + bypassDirection)
            end
        end
    end)
    
    -- Timer de segurança
    task.delay(60, function()
        if isTeleportingToBase then
            isTeleportingToBase = false
            
            if humanoidRootPart and humanoidRootPart.Parent then
                local bv = humanoidRootPart:FindFirstChildOfClass("BodyVelocity")
                local bg = humanoidRootPart:FindFirstChildOfClass("BodyGyro")
                
                if bv then bv:Destroy() end
                if bg then bg:Destroy() end
                
                humanoidRootPart.CFrame = CFrame.new(targetPosition)
                humanoid.WalkSpeed = originalWalkSpeed
                humanoid.JumpPower = originalJumpPower
            end
            
            if guideEffect and guideEffect.Parent then
                guideEffect:Destroy()
            end
            
            if teleportConnection then
                teleportConnection:Disconnect()
                teleportConnection = nil
            end
        end
    end)
    
    return true
end

-- Função para cancelar teleporte
local function cancelTeleportToBase()
    if isTeleportingToBase then
        isTeleportingToBase = false
        
        local character = player.Character
        if character then
            local humanoid = character:FindFirstChildOfClass("Humanoid")
            local humanoidRootPart = character:FindFirstChild("HumanoidRootPart")
            
            if humanoid then
                humanoid.WalkSpeed = 16
                humanoid.JumpPower = 50
            end
            
            if humanoidRootPart then
                local bv = humanoidRootPart:FindFirstChildOfClass("BodyVelocity")
                local bg = humanoidRootPart:FindFirstChildOfClass("BodyGyro")
                
                if bv then bv:Destroy() end
                if bg then bg:Destroy() end
            end
        end
        
        -- Limpar efeitos
        for _, obj in pairs(workspace:GetChildren()) do
            if obj.Name == "TeleportGuide" then
                obj:Destroy()
            end
        end
        
        if teleportConnection then
            teleportConnection:Disconnect()
            teleportConnection = nil
        end
        
        return true
    end
    return false
end

-- Função toggle para o botão
local function toggleTeleportToBase()
    if isTeleportingToBase then
        return cancelTeleportToBase()
    else
        return teleportToBaseTeleguided()
    end
end

-- [RESTANTE DO CÓDIGO DO HUB...]
-- MAIN FRAME
local main = Instance.new("Frame")
main.Size = UDim2.fromOffset(280, 420)
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
title.Text = "DEVOURER HUB"
title.TextColor3 = Color3.fromRGB(255, 255, 255)
title.Font = Enum.Font.GothamBlack
title.TextSize = 20
title.TextStrokeTransparency = 0.7
title.Parent = header

local subtitle = Instance.new("TextLabel")
subtitle.Size = UDim2.new(1, 0, 0, 20)
subtitle.Position = UDim2.fromOffset(0, 28)
subtitle.BackgroundTransparency = 1
subtitle.Text = "v3.1.7 | INSERT TO HIDE"
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
local function createButton(text, func)
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
		func = func
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
		
		-- Se for TP TO BASE, alterna entre ativar/cancelar
		if text == "TP TO BASE" then
			local wasActive = btnData.isActive
			btnData.isActive = toggleTeleportToBase()
			
			if btnData.isActive ~= wasActive then
				if btnData.isActive then
					-- Ativado (teleporte em andamento)
					TweenService:Create(button, TweenInfo.new(0.2), {
						BackgroundColor3 = Color3.fromRGB(30, 60, 120)
					}):Play()
					TweenService:Create(stroke, TweenInfo.new(0.2), {
						Color = Color3.fromRGB(0, 170, 255),
						Thickness = 2
					}):Play()
					TweenService:Create(toggleCircle, TweenInfo.new(0.2), {
						BackgroundColor3 = Color3.fromRGB(0, 170, 255),
						Size = UDim2.new(0, 24, 0, 24)
					}):Play()
					TweenService:Create(buttonText, TweenInfo.new(0.2), {
						TextColor3 = Color3.fromRGB(255, 255, 255)
					}):Play()
					buttonText.Text = "TP TO BASE (ATIVO)"
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
					buttonText.Text = "TP TO BASE"
				end
			end
			return
		end
		
		-- Para outros botões, toggle normal
		btnData.isActive = not btnData.isActive
		
		if btnData.isActive then
			-- Ativado (Verde)
			TweenService:Create(button, TweenInfo.new(0.2), {
				BackgroundColor3 = Color3.fromRGB(30, 60, 30)
			}):Play()
			TweenService:Create(stroke, TweenInfo.new(0.2), {
				Color = Color3.fromRGB(0, 180, 0),
				Thickness = 2
			}):Play()
			TweenService:Create(toggleCircle, TweenInfo.new(0.2), {
				BackgroundColor3 = Color3.fromRGB(0, 220, 0),
				Size = UDim2.new(0, 24, 0, 24)
			}):Play()
			TweenService:Create(buttonText, TweenInfo.new(0.2), {
				TextColor3 = Color3.fromRGB(255, 255, 255)
			}):Play()
			
			-- Executar função
			if func then
				func()
			end
			
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

-- Função Desync V3 (sem prints)
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

-- Funções para os outros botões
local function speedInfJump()
    -- Implemente aqui
end

local function flyV2()
    -- Implemente aqui
end

local function tpToBest()
    -- Implemente aqui
end

local function stealFloor()
    -- Implemente aqui
end

-- CRIAR APENAS 2 CATEGORIAS
task.wait(0.1)

-- CATEGORIA 1: MOVEMENT
createSection("MOVEMENT")
createButton("DESYNC V3", activateDesync)
createButton("SPEED INF JUMP", speedInfJump)
createButton("FLY V2", flyV2)

-- CATEGORIA 2: TELEPORT
createSection("TELEPORT")
createButton("TP TO BASE", toggleTeleportToBase)  -- Agora teleguiado DIRETO
createButton("TP TO BEST", tpToBest)
createButton("STEAL FLOOR", stealFloor)

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
	Size = UDim2.fromOffset(280, 420)
}):Play()

-- Controle com Insert
UserInputService.InputBegan:Connect(function(input, gameProcessed)
	if not gameProcessed and input.KeyCode == Enum.KeyCode.Insert then
		gui.Enabled = not gui.Enabled
	end
end)
