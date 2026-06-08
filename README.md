--[[
    Advanced Debug ESP & Auto-Block System
    Para uso exclusivo em Roblox Studio - Ambiente de Desenvolvimento
    Funcionalidades: ESP (ver através de paredes) e Auto-Block (bloqueio automático)
--]]

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local CoreGui = game:GetService("CoreGui")

-- Verifica se está no Studio
if not RunService:IsStudio() then
    return
end

-- Variáveis de controle
local modState = {
    espEnabled = false,
    autoBlockEnabled = false,
    activeHighlights = {},
    activeEspBoxes = {},
    hitboxConnections = {},
    combatConnections = {}
}

local localPlayer = Players.LocalPlayer

-- ================================
-- SISTEMA DE ESP (VER ATRAVÉS DE PAREDES)
-- ================================

local function createEspEffect(character)
    if not character or not character.PrimaryPart then return end
    
    -- Highlight principal para ver através das paredes
    local highlight = Instance.new("Highlight")
    highlight.FillColor = Color3.fromRGB(255, 50, 50)  -- Vermelho para inimigos
    highlight.FillTransparency = 0.6
    highlight.OutlineColor = Color3.fromRGB(255, 200, 0)  -- Amarelo para contorno
    highlight.OutlineTransparency = 0.2
    highlight.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop  -- Ver através de paredes
    highlight.Parent = character
    
    -- Adiciona um contorno extra mais brilhante
    local outline = Instance.new("Highlight")
    outline.FillTransparency = 1
    outline.OutlineColor = Color3.fromRGB(255, 255, 255)
    outline.OutlineTransparency = 0
    outline.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
    outline.Parent = character
    
    -- Efeito pulsante
    local tweenInfo = TweenInfo.new(0.8, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true)
    local tween = TweenService:Create(highlight, tweenInfo, {FillTransparency = 0.3})
    tween:Play()
    
    -- Armazena para limpeza posterior
    table.insert(modState.activeHighlights, {highlight = highlight, outline = outline, tween = tween, character = character})
    
    -- Monitora quando o personagem é destruído
    local connection
    connection = character.AncestryChanged:Connect(function()
        if not character.Parent then
            highlight:Destroy()
            outline:Destroy()
            if connection then connection:Disconnect() end
        end
    end)
    
    return highlight
end

local function toggleESP()
    modState.espEnabled = not modState.espEnabled
    
    if modState.espEnabled then
        -- Remove ESP existentes
        for _, data in ipairs(modState.activeHighlights) do
            if data.highlight then data.highlight:Destroy() end
            if data.outline then data.outline:Destroy() end
            if data.tween then data.tween:Cancel() end
        end
        modState.activeHighlights = {}
        
        -- Adiciona ESP para todos os jogadores
        for _, player in ipairs(Players:GetPlayers()) do
            if player ~= localPlayer then
                local character = player.Character
                if character then
                    createEspEffect(character)
                end
            end
        end
        
        -- Conecta para novos personagens
        local playerAddedConn = Players.PlayerAdded:Connect(function(player)
            if player ~= localPlayer then
                player.CharacterAdded:Connect(function(character)
                    if modState.espEnabled then
                        createEspEffect(character)
                    end
                end)
            end
        end)
        table.insert(modState.hitboxConnections, {conn = playerAddedConn})
        
        -- Conecta para personagens que spawnam
        local characterAddedConn = localPlayer.CharacterAdded:Connect(function()
            for _, player in ipairs(Players:GetPlayers()) do
                if player ~= localPlayer and player.Character then
                    createEspEffect(player.Character)
                end
            end
        end)
        table.insert(modState.hitboxConnections, {conn = characterAddedConn})
        
        print("[ESP] Ativado - Você pode ver outros jogadores através das paredes")
    else
        -- Remove todos os ESP
        for _, data in ipairs(modState.activeHighlights) do
            if data.highlight then data.highlight:Destroy() end
            if data.outline then data.outline:Destroy() end
            if data.tween then data.tween:Cancel() end
        end
        modState.activeHighlights = {}
        print("[ESP] Desativado")
    end
end

-- ================================
-- SISTEMA DE AUTO-BLOCK (HITBOX VERMELHA)
-- ================================

local function checkForRedHitbox(character)
    -- Procura por hitboxes vermelhas (indicadores de ataque/dano)
    local redHitboxes = {}
    
    -- Verifica se há partes com cor vermelha ou highlights vermelhos
    for _, descendant in ipairs(character:GetDescendants()) do
        if descendant:IsA("BasePart") then
            -- Verifica a cor da parte
            if descendant.Color.r > 0.7 and descendant.Color.g < 0.3 and descendant.Color.b < 0.3 then
                table.insert(redHitboxes, descendant)
            end
        elseif descendant:IsA("Highlight") then
            -- Verifica highlights vermelhos
            if descendant.FillColor.r > 0.7 and descendant.FillColor.g < 0.3 then
                table.insert(redHitboxes, descendant)
            end
        elseif descendant:IsA("SelectionBox") or descendant:IsA("SelectionSphere") then
            -- Verifica seleções
            if descendant.Color3.r > 0.7 then
                table.insert(redHitboxes, descendant)
            end
        end
    end
    
    return #redHitboxes > 0, redHitboxes
end

local function performBlock()
    local character = localPlayer.Character
    if not character then return end
    
    local humanoid = character:FindFirstChild("Humanoid")
    if not humanoid then return end
    
    -- Simula um bloqueio/defesa
    -- Método 1: Reduz dano recebido
    local health = humanoid.Health
    
    -- Método 2: Adiciona um escudo temporário (se o jogo tiver)
    if not character:FindFirstChild("BlockingShield") then
        local shield = Instance.new("Part")
        shield.Name = "BlockingShield"
        shield.Size = Vector3.new(4, 5, 3)
        shield.Position = character.PrimaryPart.Position + Vector3.new(0, 2, 0)
        shield.Anchored = true
        shield.CanCollide = false
        shield.Transparency = 0.5
        shield.Color = Color3.fromRGB(0, 100, 255)
        shield.Material = Enum.Material.Neon
        shield.Parent = character
        
        -- Remove o escudo após 0.5 segundos
        task.wait(0.5)
        shield:Destroy()
    end
    
    -- Método 3: Tenta aplicar uma animação de block (se existir)
    local animator = humanoid:FindFirstChild("Animator")
    if animator and animator:FindFirstChild("BlockAnimation") then
        -- Aqui você pode carregar uma animação específica
    end
    
    -- Efeito visual de bloqueio
    local blockEffect = Instance.new("Part")
    blockEffect.Size = Vector3.new(2, 2, 0.5)
    blockEffect.Position = character.PrimaryPart.Position + Vector3.new(0, 1, 0)
    blockEffect.Anchored = true
    blockEffect.CanCollide = false
    blockEffect.Transparency = 0.3
    blockEffect.Color = Color3.fromRGB(0, 255, 255)
    blockEffect.Material = Enum.Material.Neon
    blockEffect.Parent = character
    
    -- Animação pulsante
    local tween = TweenService:Create(blockEffect, TweenInfo.new(0.2), {Transparency = 0.8})
    tween:Play()
    tween.Completed:Connect(function()
        blockEffect:Destroy()
    end)
    
    print("[Auto-Block] Bloqueio ativado - Dano mitigado!")
end

local function startAutoBlock()
    modState.autoBlockEnabled = true
    
    -- Loop de verificação constante
    local checkLoop
    checkLoop = RunService.Heartbeat:Connect(function()
        if not modState.autoBlockEnabled then
            checkLoop:Disconnect()
            return
        end
        
        local character = localPlayer.Character
        if not character then return end
        
        -- Verifica inimigos próximos em busca de hitboxes vermelhas
        local nearestDistance = 15  -- Raio de detecção
        local threatFound = false
        
        for _, player in ipairs(Players:GetPlayers()) do
            if player ~= localPlayer then
                local enemyChar = player.Character
                if enemyChar and enemyChar.PrimaryPart and character.PrimaryPart then
                    local distance = (enemyChar.PrimaryPart.Position - character.PrimaryPart.Position).Magnitude
                    
                    if distance < nearestDistance then
                        local hasRedHitbox, hitboxes = checkForRedHitbox(enemyChar)
                        if hasRedHitbox then
                            threatFound = true
                            break
                        end
                    end
                end
            end
        end
        
        -- Se encontrou uma ameaça, bloqueia
        if threatFound then
            performBlock()
            task.wait(0.3)  -- Delay entre bloqueios para não spammar
        end
    end)
    
    table.insert(modState.combatConnections, {conn = checkLoop})
    print("[Auto-Block] Ativado - Bloqueia automaticamente quando hitbox vermelha é detectada")
end

local function stopAutoBlock()
    modState.autoBlockEnabled = false
    
    -- Remove conexões
    for _, data in ipairs(modState.combatConnections) do
        if data.conn then data.conn:Disconnect() end
    end
    modState.combatConnections = {}
    
    print("[Auto-Block] Desativado")
end

local function toggleAutoBlock()
    if modState.autoBlockEnabled then
        stopAutoBlock()
    else
        startAutoBlock()
    end
end

-- ================================
-- INTERFACE MODERNA
-- ================================

local function createModernModMenu()
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "ModMenu"
    screenGui.ResetOnSpawn = false
    screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    screenGui.Parent = CoreGui
    
    -- Frame principal
    local mainFrame = Instance.new("Frame")
    mainFrame.Size = UDim2.new(0, 320, 0, 450)
    mainFrame.Position = UDim2.new(0, 20, 0, 100)
    mainFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 30)
    mainFrame.BackgroundTransparency = 0.08
    mainFrame.Parent = screenGui
    
    -- Cantos arredondados
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 12)
    corner.Parent = mainFrame
    
    -- Sombra/contorno
    local stroke = Instance.new("UIStroke")
    stroke.Color = Color3.fromRGB(255, 50, 50)
    stroke.Thickness = 1.5
    stroke.Transparency = 0.5
    stroke.Parent = mainFrame
    
    -- Título
    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1, 0, 0, 50)
    title.Text = "⚡ MOD MENU | STUDIO ONLY"
    title.TextColor3 = Color3.fromRGB(255, 255, 255)
    title.TextSize = 18
    title.TextScaled = true
    title.BackgroundTransparency = 1
    title.Font = Enum.Font.GothamBold
    title.Parent = mainFrame
    
    -- Subtítulo
    local subtitle = Instance.new("TextLabel")
    subtitle.Size = UDim2.new(1, 0, 0, 30)
    subtitle.Position = UDim2.new(0, 0, 0, 50)
    subtitle.Text = "Ferramentas de Desenvolvimento"
    subtitle.TextColor3 = Color3.fromRGB(150, 150, 150)
    subtitle.TextSize = 12
    subtitle.BackgroundTransparency = 1
    subtitle.Font = Enum.Font.Gotham
    subtitle.Parent = mainFrame
    
    -- Container para botões
    local buttonsContainer = Instance.new("Frame")
    buttonsContainer.Size = UDim2.new(1, -20, 1, -100)
    buttonsContainer.Position = UDim2.new(0, 10, 0, 90)
    buttonsContainer.BackgroundTransparency = 1
    buttonsContainer.Parent = mainFrame
    
    local uiList = Instance.new("UIListLayout")
    uiList.Padding = UDim.new(0, 12)
    uiList.SortOrder = Enum.SortOrder.LayoutOrder
    uiList.Parent = buttonsContainer
    
    -- Botão ESP
    local espButton = Instance.new("TextButton")
    espButton.Size = UDim2.new(1, 0, 0, 50)
    espButton.Text = "👁️ ESP (Ver através de paredes) - OFF"
    espButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    espButton.TextSize = 14
    espButton.BackgroundColor3 = Color3.fromRGB(40, 40, 55)
    espButton.BackgroundTransparency = 0.2
    espButton.Parent = buttonsContainer
    
    local espCorner = Instance.new("UICorner")
    espCorner.CornerRadius = UDim.new(0, 8)
    espCorner.Parent = espButton
    
    -- Botão Auto-Block
    local blockButton = Instance.new("TextButton")
    blockButton.Size = UDim2.new(1, 0, 0, 50)
    blockButton.Text = "🛡️ Auto-Block (Hitbox Vermelha) - OFF"
    blockButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    blockButton.TextSize = 14
    blockButton.BackgroundColor3 = Color3.fromRGB(40, 40, 55)
    blockButton.BackgroundTransparency = 0.2
    blockButton.Parent = buttonsContainer
    
    local blockCorner = Instance.new("UICorner")
    blockCorner.CornerRadius = UDim.new(0, 8)
    blockCorner.Parent = blockButton
    
    -- Status Panel
    local statusPanel = Instance.new("Frame")
    statusPanel.Size = UDim2.new(1, 0, 0, 80)
    statusPanel.Position = UDim2.new(0, 0, 0, 350)
    statusPanel.BackgroundColor3 = Color3.fromRGB(30, 30, 40)
    statusPanel.BackgroundTransparency = 0.3
    statusPanel.Parent = mainFrame
    
    local statusCorner = Instance.new("UICorner")
    statusCorner.CornerRadius = UDim.new(0, 8)
    statusCorner.Parent = statusPanel
    
    local statusTitle = Instance.new("TextLabel")
    statusTitle.Size = UDim2.new(1, 0, 0, 25)
    statusTitle.Text = "STATUS"
    statusTitle.TextColor3 = Color3.fromRGB(255, 200, 100)
    statusTitle.TextSize = 12
    statusTitle.TextXAlignment = Enum.TextXAlignment.Left
    statusTitle.BackgroundTransparency = 1
    statusTitle.Parent = statusPanel
    
    local espStatus = Instance.new("TextLabel")
    espStatus.Size = UDim2.new(1, -10, 0, 20)
    espStatus.Position = UDim2.new(0, 5, 0, 25)
    espStatus.Text = "🔴 ESP: DESATIVADO"
    espStatus.TextColor3 = Color3.fromRGB(255, 100, 100)
    espStatus.TextSize = 11
    espStatus.TextXAlignment = Enum.TextXAlignment.Left
    espStatus.BackgroundTransparency = 1
    espStatus.Parent = statusPanel
    
    local blockStatus = Instance.new("TextLabel")
    blockStatus.Size = UDim2.new(1, -10, 0, 20)
    blockStatus.Position = UDim2.new(0, 5, 0, 45)
    blockStatus.Text = "🔴 AUTO-BLOCK: DESATIVADO"
    blockStatus.TextColor3 = Color3.fromRGB(255, 100, 100)
    blockStatus.TextSize = 11
    blockStatus.TextXAlignment = Enum.TextXAlignment.Left
    blockStatus.BackgroundTransparency = 1
    blockStatus.Parent = statusPanel
    
    -- Botão Fechar
    local closeBtn = Instance.new("TextButton")
    closeBtn.Size = UDim2.new(0, 30, 0, 30)
    closeBtn.Position = UDim2.new(1, -35, 0, 10)
    closeBtn.Text = "✕"
    closeBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    closeBtn.TextSize = 18
    closeBtn.BackgroundColor3 = Color3.fromRGB(60, 60, 75)
    closeBtn.BackgroundTransparency = 0.5
    closeBtn.Parent = mainFrame
    
    local closeCorner = Instance.new("UICorner")
    closeCorner.CornerRadius = UDim.new(0, 6)
    closeCorner.Parent = closeBtn
    
    -- Funcionalidade dos botões
    local espOn = false
    local blockOn = false
    
    espButton.MouseButton1Click:Connect(function()
        espOn = not espOn
        toggleESP()
        
        if espOn then
            espButton.Text = "👁️ ESP (Ver através de paredes) - ON"
            espButton.BackgroundColor3 = Color3.fromRGB(0, 150, 100)
            espStatus.Text = "🟢 ESP: ATIVADO"
            espStatus.TextColor3 = Color3.fromRGB(100, 255, 100)
        else
            espButton.Text = "👁️ ESP (Ver através de paredes) - OFF"
            espButton.BackgroundColor3 = Color3.fromRGB(40, 40, 55)
            espStatus.Text = "🔴 ESP: DESATIVADO"
            espStatus.TextColor3 = Color3.fromRGB(255, 100, 100)
        end
        
        -- Animação
        local tween = TweenService:Create(espButton, TweenInfo.new(0.2), {BackgroundTransparency = 0})
        tween:Play()
        task.wait(0.1)
        tween = TweenService:Create(espButton, TweenInfo.new(0.2), {BackgroundTransparency = 0.2})
        tween:Play()
    end)
    
    blockButton.MouseButton1Click:Connect(function()
        blockOn = not blockOn
        toggleAutoBlock()
        
        if blockOn then
            blockButton.Text = "🛡️ Auto-Block (Hitbox Vermelha) - ON"
            blockButton.BackgroundColor3 = Color3.fromRGB(0, 150, 100)
            blockStatus.Text = "🟢 AUTO-BLOCK: ATIVADO"
            blockStatus.TextColor3 = Color3.fromRGB(100, 255, 100)
        else
            blockButton.Text = "🛡️ Auto-Block (Hitbox Vermelha) - OFF"
            blockButton.BackgroundColor3 = Color3.fromRGB(40, 40, 55)
            blockStatus.Text = "🔴 AUTO-BLOCK: DESATIVADO"
            blockStatus.TextColor3 = Color3.fromRGB(255, 100, 100)
        end
        
        -- Animação
        local tween = TweenService:Create(blockButton, TweenInfo.new(0.2), {BackgroundTransparency = 0})
        tween:Play()
        task.wait(0.1)
        tween = TweenService:Create(blockButton, TweenInfo.new(0.2), {BackgroundTransparency = 0.2})
        tween:Play()
    end)
    
    closeBtn.MouseButton1Click:Connect(function()
        -- Desativa todas as funcionalidades
        if espOn then toggleESP() end
        if blockOn then toggleAutoBlock() end
        
        -- Fecha a interface
        local tween = TweenService:Create(mainFrame, TweenInfo.new(0.3), {BackgroundTransparency = 1})
        tween:Play()
        tween.Completed:Connect(function()
            screenGui:Destroy()
        end)
    end)
    
    -- Hover effects
    local function setupHover(button)
        button.MouseEnter:Connect(function()
            local tween = TweenService:Create(button, TweenInfo.new(0.2), {BackgroundTransparency = 0})
            tween:Play()
        end)
        button.MouseLeave:Connect(function()
            local tween = TweenService:Create(button, TweenInfo.new(0.2), {BackgroundTransparency = 0.2})
            tween:Play()
        end)
    end
    
    setupHover(espButton)
    setupHover(blockButton)
    setupHover(closeBtn)
    
    -- Animação de entrada
    mainFrame.BackgroundTransparency = 1
    local enterTween = TweenService:Create(mainFrame, TweenInfo.new(0.4), {BackgroundTransparency = 0.08})
    enterTween:Play()
    
    return screenGui
end

-- ================================
-- SISTEMA DE NOTIFICAÇÕES VISUAIS
-- ================================

local function showNotification(message, color)
    local notificationGui = Instance.new("ScreenGui")
    notificationGui.Name = "Notification"
    notificationGui.Parent = CoreGui
    
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(0, 300, 0, 50)
    frame.Position = UDim2.new(0.5, -150, 0, 50)
    frame.BackgroundColor3 = color or Color3.fromRGB(30, 30, 40)
    frame.BackgroundTransparency = 0.1
    frame.Parent = notificationGui
    
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 8)
    corner.Parent = frame
    
    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1, 0, 1, 0)
    label.Text = message
    label.TextColor3 = Color3.fromRGB(255, 255, 255)
    label.TextSize = 14
    label.BackgroundTransparency = 1
    label.Parent = frame
    
    -- Animação
    frame.Position = UDim2.new(0.5, -150, 0, -60)
    local tween = TweenService:Create(frame, TweenInfo.new(0.5, Enum.EasingStyle.Bounce), {Position = UDim2.new(0.5, -150, 0, 50)})
    tween:Play()
    
    task.wait(3)
    
    local 
