--====================================================
-- GOJO SMART KAITUN V8 - RECUPERACAO COM ROTA AEREA
-- Mantem GUI, contador, perseguicao, combo e recuperacao.
-- Defesa limitada a M1; nao inclui defesa contra habilidades.
-- OBS: movimento feito pelo cliente pode sofrer rollback do servidor.
--====================================================

local Players = game:GetService("Players")
local TweenService = game:GetService("TweenService")
local UIS = game:GetService("UserInputService")
local VirtualInputManager = game:GetService("VirtualInputManager")
local RunService = game:GetService("RunService")
local RS = game:GetService("ReplicatedStorage")
local LP = Players.LocalPlayer

-- CONFIG
local GOAL = 1000
local M1_DELAY = 0.40
-- Habilidades ativadas pelas teclas 1-4; cooldowns em segundos.
local ABILITIES = {
    {key = Enum.KeyCode.One,   name = "Lapse Blue",       cooldown = 13},
    {key = Enum.KeyCode.Two,   name = "Rapid Punches",    cooldown = 15},
    {key = Enum.KeyCode.Three, name = "Reversal Red",     cooldown = 20},
    {key = Enum.KeyCode.Four,  name = "Twofold Kick",     cooldown = 18},
}
local LastAbilityUse = {}
local ABILITY_KEY_HOLD = 0.08
local ABILITY_GAP = 0.18
local TWEEN_SPEED = 100
local CHASE_REFRESH = 0.18
local REPATH_DISTANCE = 2.5
local TARGET_DISTANCE = 2.8
local ATTACK_DISTANCE = 5.2
local BLOCK_DISTANCE = 8
local MAX_TARGET_DISTANCE = 1000
local LOOP_DELAY = 0.02

-- HP / LOJA
local LOW_HP_PERCENT = 0.25
local RECOVER_TO_PERCENT = 0.75
local SHOP_DISTANCE = 7
local SHOP_TWEEN_SPEED = 115
local SHOP_FLIGHT_HEIGHT = 38 -- altura extra para atravessar por cima de obstáculos
local SHOP_INTERACT_TRIES = 5
local RECOVERY_WAIT_LIMIT = 30
local M1_DEFENSE_TIMEOUT = 0.85

local DRINK_NAMES = {"soda", "cola", "drink", "refrigerante", "vending", "beverage", "pop", "can", "machine"}
local TOOL_DRINK_NAMES = {"soda", "cola", "drink", "refrigerante", "beverage", "pop"}

-- STATE
local Running = false
local CurrentTarget = nil
local Blocking = false
local BusyCombo = false
local CurrentTween = nil
local CurrentTweenGoal = nil
local LastTweenUpdate = 0
local TargetConnections = {}
local FullM1Defense = false
local FourthOnlyDefense = false
local LastTargetM1 = nil
local LastM1ChangeTime = 0
local DefenseGeneration = 0
local Recovering = false
local RecoveryGeneration = 0
local ScriptGeneration = 0
local disconnectTargetMonitor -- declarado antes da recuperação para bloquear eventos antigos
local BlockFaceConnection = nil
local CombatFaceConnection = nil

-- NOCLIP SOMENTE DURANTE A RECUPERACAO; restaura colisao original ao terminar/parar.
local NoclipEnabled = false
local NoclipConnection = nil
local NoclipCharacterConnection = nil
local OriginalCollision = {}

local function restoreNoclip()
    NoclipEnabled = false
    if NoclipConnection then
        NoclipConnection:Disconnect()
        NoclipConnection = nil
    end
    for part, original in pairs(OriginalCollision) do
        if part and part.Parent and part:IsA("BasePart") then
            part.CanCollide = original
        end
    end
    table.clear(OriginalCollision)
end

local function applyNoclip()
    local character = LP.Character
    if not character then return end
    for _, part in ipairs(character:GetDescendants()) do
        if part:IsA("BasePart") then
            if OriginalCollision[part] == nil then
                OriginalCollision[part] = part.CanCollide
            end
            part.CanCollide = false
        end
    end
end

local function startNoclip()
    if NoclipEnabled then return end
    NoclipEnabled = true
    applyNoclip()
    NoclipConnection = RunService.Stepped:Connect(function()
        if NoclipEnabled then applyNoclip() end
    end)
end

NoclipCharacterConnection = LP.CharacterAdded:Connect(function()
    restoreNoclip()
    if Recovering and Running then startNoclip() end
end)

-- Encerra a instancia anterior caso o script seja executado novamente.
local SharedEnv = (type(getgenv) == "function" and getgenv()) or _G
if type(SharedEnv.GojoSmartKaitunStop) == "function" then
    pcall(SharedEnv.GojoSmartKaitunStop)
end
SharedEnv.GojoSmartKaitunStop = nil

-- KNIT
local Knit = require(RS:WaitForChild("Knit"):WaitForChild("Knit"))
local GojoService = Knit.GetService("GojoService")
local BlockService = Knit.GetService("BlockService")

-- KILLS / CHARACTERS
local leaderstats = LP:WaitForChild("leaderstats")
local KillsValue = leaderstats:FindFirstChild("Kills", true) or leaderstats:FindFirstChild("kills", true)
if not KillsValue then warn("Kills não encontradas."); return end
local Characters = workspace:WaitForChild("Characters")

-- GUI
local PlayerGui = LP:WaitForChild("PlayerGui")
local old = PlayerGui:FindFirstChild("GojoSmartKaitunV8")
if old then old:Destroy() end

local gui = Instance.new("ScreenGui")
gui.Name = "GojoSmartKaitunV8"
gui.ResetOnSpawn = false
gui.IgnoreGuiInset = true
gui.DisplayOrder = 999999
gui.Parent = PlayerGui

local frame = Instance.new("Frame")
frame.Size = UDim2.fromOffset(345, 455)
frame.Position = UDim2.new(0.5, -172, 0.12, 0)
frame.BackgroundColor3 = Color3.fromRGB(18, 18, 18)
frame.BorderSizePixel = 0
frame.Parent = gui
Instance.new("UICorner", frame).CornerRadius = UDim.new(0, 12)

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, -50, 0, 40)
title.Position = UDim2.fromOffset(10, 0)
title.BackgroundTransparency = 1
title.Text = "GOJO SMART KAITUN V8"
title.TextColor3 = Color3.new(1, 1, 1)
title.Font = Enum.Font.GothamBold
title.TextSize = 17
title.TextXAlignment = Enum.TextXAlignment.Left
title.Active = true
title.Parent = frame

local close = Instance.new("TextButton")
close.Size = UDim2.fromOffset(36, 32)
close.Position = UDim2.new(1, -42, 0, 4)
close.BackgroundColor3 = Color3.fromRGB(55, 55, 55)
close.TextColor3 = Color3.new(1, 1, 1)
close.Text = "X"
close.Font = Enum.Font.GothamBold
close.TextSize = 16
close.Parent = frame
Instance.new("UICorner", close).CornerRadius = UDim.new(0, 8)

local killsLabel = Instance.new("TextLabel")
killsLabel.Size = UDim2.new(1, -20, 0, 40)
killsLabel.Position = UDim2.fromOffset(10, 45)
killsLabel.BackgroundColor3 = Color3.fromRGB(8, 8, 8)
killsLabel.TextColor3 = Color3.new(1, 1, 1)
killsLabel.Font = Enum.Font.GothamBold
killsLabel.TextSize = 18
killsLabel.Parent = frame
Instance.new("UICorner", killsLabel).CornerRadius = UDim.new(0, 8)

local hpLabel = Instance.new("TextLabel")
hpLabel.Size = UDim2.new(1, -20, 0, 35)
hpLabel.Position = UDim2.fromOffset(10, 92)
hpLabel.BackgroundColor3 = Color3.fromRGB(8, 8, 8)
hpLabel.TextColor3 = Color3.new(1, 1, 1)
hpLabel.Font = Enum.Font.GothamBold
hpLabel.TextSize = 14
hpLabel.Text = "HP: ?"
hpLabel.Parent = frame
Instance.new("UICorner", hpLabel).CornerRadius = UDim.new(0, 8)

local targetLabel = Instance.new("TextLabel")
targetLabel.Size = UDim2.new(1, -20, 0, 75)
targetLabel.Position = UDim2.fromOffset(10, 134)
targetLabel.BackgroundColor3 = Color3.fromRGB(8, 8, 8)
targetLabel.TextColor3 = Color3.new(1, 1, 1)
targetLabel.Font = Enum.Font.Code
targetLabel.TextSize = 13
targetLabel.TextWrapped = true
targetLabel.Text = "Alvo: nenhum"
targetLabel.Parent = frame
Instance.new("UICorner", targetLabel).CornerRadius = UDim.new(0, 8)

local status = Instance.new("TextLabel")
status.Size = UDim2.new(1, -20, 0, 75)
status.Position = UDim2.fromOffset(10, 217)
status.BackgroundColor3 = Color3.fromRGB(8, 8, 8)
status.TextColor3 = Color3.new(1, 1, 1)
status.Font = Enum.Font.Code
status.TextSize = 13
status.TextWrapped = true
status.Text = "Status: parado"
status.Parent = frame
Instance.new("UICorner", status).CornerRadius = UDim.new(0, 8)

local toggle = Instance.new("TextButton")
toggle.Size = UDim2.new(1, -20, 0, 43)
toggle.Position = UDim2.fromOffset(10, 405)
toggle.BackgroundColor3 = Color3.fromRGB(45, 45, 45)
toggle.TextColor3 = Color3.new(1, 1, 1)
toggle.Font = Enum.Font.GothamBold
toggle.TextSize = 17
toggle.Text = "INICIAR"
toggle.Parent = frame
Instance.new("UICorner", toggle).CornerRadius = UDim.new(0, 8)

-- OCULTAR CONSTRUCOES LOCALMENTE (nao apaga objetos do servidor)
-- Mantem visiveis: maquinas de refrigerante, personagens/NPCs e superficies de chao.
local mapHidden = false
local savedLocalTransparency = {}
local mapAddedConnection = nil

local MACHINE_KEYWORDS = {"soda", "cola", "drink", "refrigerante", "vending", "beverage", "pop", "machine"}
local GROUND_KEYWORDS = {"ground", "floor", "chao", "baseplate", "terrain", "road", "street", "path", "platform", "piso", "asphalt", "calçada", "calcada"}

local function nameHasKeyword(name, keywords)
    name = string.lower(name or "")
    for _, word in ipairs(keywords) do
        if string.find(name, word, 1, true) then return true end
    end
    return false
end

local function belongsToMachine(part)
    local node = part
    while node and node ~= workspace do
        if nameHasKeyword(node.Name, MACHINE_KEYWORDS) then return true end
        node = node.Parent
    end
    return false
end

local function belongsToCharacter(part)
    local node = part
    while node and node ~= workspace do
        if node:IsA("Model") and node:FindFirstChildOfClass("Humanoid") then return true end
        node = node.Parent
    end
    return false
end

local function isGroundSurface(part)
    local n = string.lower(part.Name)
    if nameHasKeyword(n, GROUND_KEYWORDS) then return true end
    -- Heuristica para pisos grandes e finos, evitando esconder o chao principal.
    local size = part.Size
    local largest = math.max(size.X, size.Z)
    return largest >= 45 and size.Y <= 8
end

local function shouldKeepVisible(part)
    return belongsToMachine(part) or belongsToCharacter(part) or isGroundSurface(part)
end

local function hideOneMapPart(inst)
    if not mapHidden or not inst:IsA("BasePart") or shouldKeepVisible(inst) then return end
    if savedLocalTransparency[inst] == nil then
        savedLocalTransparency[inst] = inst.LocalTransparencyModifier
    end
    inst.LocalTransparencyModifier = 1
end

local function restoreMapVisuals()
    mapHidden = false
    if mapAddedConnection then
        mapAddedConnection:Disconnect()
        mapAddedConnection = nil
    end
    for part, original in pairs(savedLocalTransparency) do
        if part and part.Parent and part:IsA("BasePart") then
            part.LocalTransparencyModifier = original
        end
    end
    table.clear(savedLocalTransparency)
end

local function hideMapVisuals()
    if mapHidden then return end
    mapHidden = true
    for _, inst in ipairs(workspace:GetDescendants()) do
        hideOneMapPart(inst)
    end
    mapAddedConnection = workspace.DescendantAdded:Connect(function(inst)
        task.defer(function()
            if mapHidden and inst.Parent then hideOneMapPart(inst) end
        end)
    end)
end

local mapButton = Instance.new("TextButton")
mapButton.Size = UDim2.new(1, -20, 0, 42)
mapButton.Position = UDim2.fromOffset(10, 302)
mapButton.BackgroundColor3 = Color3.fromRGB(45, 65, 85)
mapButton.TextColor3 = Color3.new(1, 1, 1)
mapButton.Font = Enum.Font.GothamBold
mapButton.TextSize = 14
mapButton.Text = "CONSTRUÇÕES: VISÍVEIS"
mapButton.Parent = frame
Instance.new("UICorner", mapButton).CornerRadius = UDim.new(0, 8)

mapButton.MouseButton1Click:Connect(function()
    if mapHidden then
        restoreMapVisuals()
        mapButton.Text = "CONSTRUÇÕES: VISÍVEIS"
        mapButton.BackgroundColor3 = Color3.fromRGB(45, 65, 85)
    else
        hideMapVisuals()
        mapButton.Text = "CONSTRUÇÕES: OCULTAS"
        mapButton.BackgroundColor3 = Color3.fromRGB(65, 85, 45)
    end
end)

-- Os botões manuais de teclado foram removidos da interface.
-- As habilidades 1-4 continuam sendo acionadas automaticamente no combate.

local function updateKills()
    killsLabel.Text = "Kills: " .. tostring(KillsValue.Value) .. " / " .. tostring(GOAL)
end

local function getMyCharacter()
    local char = LP.Character
    if not char then return nil end
    local hum = char:FindFirstChildOfClass("Humanoid")
    local root = char:FindFirstChild("HumanoidRootPart")
    if not hum or not root or hum.Health <= 0 then return nil end
    return char, hum, root
end

local function updateHP()
    local _, hum = getMyCharacter()
    if not hum then hpLabel.Text = "HP: ?"; return end
    local pct = hum.MaxHealth > 0 and hum.Health / hum.MaxHealth or 0
    hpLabel.Text = "HP: " .. math.floor(hum.Health) .. " / " .. math.floor(hum.MaxHealth) .. " | " .. math.floor(pct * 100) .. "%"
end

local function getHumRoot(model)
    if not model or not model.Parent then return nil, nil end
    return model:FindFirstChildOfClass("Humanoid"), model:FindFirstChild("HumanoidRootPart")
end

local function targetAlive(model)
    local hum, root = getHumRoot(model)
    return hum ~= nil and root ~= nil and hum.Health > 0
end

local function healthPercent()
    local _, hum = getMyCharacter()
    if not hum or hum.MaxHealth <= 0 then return 0 end
    return hum.Health / hum.MaxHealth
end

local function getTargetType(model)
    return Players:GetPlayerFromCharacter(model) and "PLAYER" or "NPC/DUMMY"
end

local function findNearestTarget()
    local _, _, myRoot = getMyCharacter()
    if not myRoot then return nil end
    local bestPlayer, bestPlayerDist = nil, math.huge
    local bestNPC, bestNPCDist = nil, math.huge
    for _, model in ipairs(Characters:GetChildren()) do
        if model:IsA("Model") and model ~= LP.Character and targetAlive(model) then
            local _, root = getHumRoot(model)
            local dist = (root.Position - myRoot.Position).Magnitude
            if dist <= MAX_TARGET_DISTANCE then
                local p = Players:GetPlayerFromCharacter(model)
                if p and p ~= LP then
                    if dist < bestPlayerDist then bestPlayerDist, bestPlayer = dist, model end
                elseif not p and dist < bestNPCDist then
                    bestNPCDist, bestNPC = dist, model
                end
            end
        end
    end
    return bestPlayer or bestNPC
end

local function distanceTo(target)
    local _, _, root = getMyCharacter()
    local _, tr = getHumRoot(target)
    if not root or not tr then return math.huge end
    return (tr.Position - root.Position).Magnitude
end

-- MOVEMENT
local function stopTween()
    if CurrentTween then pcall(function() CurrentTween:Cancel() end) end
    CurrentTween = nil
    CurrentTweenGoal = nil
end

local function faceTarget(target)
    local _, _, root = getMyCharacter()
    local _, tr = getHumRoot(target)
    if not root or not tr then return end
    local p, t = root.Position, tr.Position
    root.CFrame = CFrame.new(p, Vector3.new(t.X, p.Y, t.Z))
end

-- Mantém a orientação atualizada mesmo enquanto a rotina de combo está aguardando.
-- Isso corrige a janela em que o alvo circulava para trás durante os waits do combo.
local function startCombatFacing()
    if CombatFaceConnection then return end
    CombatFaceConnection = RunService.RenderStepped:Connect(function()
        local target = CurrentTarget
        if Running and not Recovering and target and targetAlive(target)
            and (Blocking or distanceTo(target) <= ATTACK_DISTANCE + 1) then
            faceTarget(target)
        end
    end)
end

local function stopCombatFacing()
    if CombatFaceConnection then
        CombatFaceConnection:Disconnect()
        CombatFaceConnection = nil
    end
end

local function getChaseDestination(target)
    local _, tr = getHumRoot(target)
    if not tr then return nil end
    local desired = tr.CFrame * CFrame.new(0, 0, TARGET_DISTANCE)
    local p = Vector3.new(desired.Position.X, tr.Position.Y, desired.Position.Z)
    return CFrame.new(p, tr.Position)
end

local function updateTweenToTarget(target)
    if Recovering or not targetAlive(target) then stopTween(); return end
    local now = os.clock()
    if now - LastTweenUpdate < CHASE_REFRESH then return end
    LastTweenUpdate = now
    local _, _, root = getMyCharacter()
    if not root then return end
    local goal = getChaseDestination(target)
    if not goal then return end
    local d = (root.Position - goal.Position).Magnitude
    if d <= 0.4 then stopTween(); return end
    if CurrentTweenGoal and CurrentTween then
        if (CurrentTweenGoal.Position - goal.Position).Magnitude < REPATH_DISTANCE then return end
    end
    stopTween()
    local duration = math.max(d / TWEEN_SPEED, 0.05)
    CurrentTweenGoal = goal
    CurrentTween = TweenService:Create(root, TweenInfo.new(duration, Enum.EasingStyle.Linear), {CFrame = goal})
    CurrentTween:Play()
end

-- RECUPERACAO: sobe, atravessa horizontalmente por cima dos obstáculos e desce.
-- Cada etapa usa Tween; o servidor ainda pode corrigir a posição do cliente.
local function tweenLeg(root, destination, speed)
    if not root or not root.Parent or not Running or not Recovering then return false end
    stopTween()
    local current = root.Position
    local distance = (current - destination).Magnitude
    if distance <= 1.5 then return true end
    local goal = CFrame.new(destination, destination + root.CFrame.LookVector)
    local duration = math.max(distance / math.max(speed or SHOP_TWEEN_SPEED, 1), 0.15)
    local tween
    local ok = pcall(function()
        tween = TweenService:Create(root, TweenInfo.new(duration, Enum.EasingStyle.Linear), {CFrame = goal})
        CurrentTween = tween
        CurrentTweenGoal = goal
        tween:Play()
    end)
    if not ok or not tween then stopTween(); return false end
    local state = tween.Completed:Wait()
    if CurrentTween == tween then
        CurrentTween = nil
        CurrentTweenGoal = nil
    end
    if state ~= Enum.PlaybackState.Completed or not root.Parent or not Running or not Recovering then return false end
    task.wait(0.12)
    return root.Parent ~= nil
end

local function tweenToPosition(position, speed)
    local character, _, root = getMyCharacter()
    if not character or not root or not root.Parent then return false end
    stopTween()

    local startPosition = root.Position
    if (startPosition - position).Magnitude <= SHOP_DISTANCE then return true end

    -- Move em três etapas para não ficar apenas subindo no mesmo X/Z.
    local flightY = math.max(startPosition.Y, position.Y) + SHOP_FLIGHT_HEIGHT
    local aboveStart = Vector3.new(startPosition.X, flightY, startPosition.Z)
    local aboveShop = Vector3.new(position.X, flightY, position.Z)
    local landing = position + Vector3.new(0, 2.5, 0)

    status.Text = "RECUPERAÇÃO: SUBINDO..."
    if not tweenLeg(root, aboveStart, speed) then return false end
    status.Text = "RECUPERAÇÃO: VOANDO ATÉ A MÁQUINA..."
    if not tweenLeg(root, aboveShop, speed) then return false end
    status.Text = "RECUPERAÇÃO: DESCENDO..."
    if not tweenLeg(root, landing, math.max((speed or SHOP_TWEEN_SPEED) * 0.55, 35)) then return false end

    task.wait(0.15)
    if not root.Parent then return false end
    return (root.Position - position).Magnitude <= SHOP_DISTANCE
end

-- BLOCK
local function startBlock(target)
    if not targetAlive(target) then return end
    if not Blocking then
        stopTween()
        local ok, err = pcall(function() BlockService.Activated:Fire(target) end)
        if ok then
            Blocking = true
        else
            warn("Falha ao iniciar Block:", err)
            return
        end
    end

    -- Continua virando para o alvo durante a defesa, inclusive se ele circular por trás.
    faceTarget(target)
    if not BlockFaceConnection then
        BlockFaceConnection = RunService.RenderStepped:Connect(function()
            if Blocking and Running and not Recovering and target == CurrentTarget and targetAlive(target) then
                faceTarget(target)
            end
        end)
    end
end

local function stopBlock()
    if BlockFaceConnection then
        BlockFaceConnection:Disconnect()
        BlockFaceConnection = nil
    end
    if not Blocking then return end
    pcall(function() BlockService.Deactivated:Fire() end)
    Blocking = false
end

-- ATTACK
local function getAttackType()
    local _, hum = getMyCharacter()
    if not hum then return nil end
    if hum.FloorMaterial == Enum.Material.Air then return "Down" end
    if hum.Jump then return "Up" end
    return nil
end

local function oneM1(target)
    if not targetAlive(target) then return false end
    local ok, err = pcall(function() GojoService.Activated:Fire(getAttackType(), target) end)
    if not ok then warn("Falha no M1:", err) end
    return ok
end

local function amStunned()
    local char = LP.Character
    if not char then return true end
    local info = char:FindFirstChild("Info")
    local stun = info and info:FindFirstChild("Stun")
    if stun then
        if stun:IsA("BoolValue") then return stun.Value end
        if stun:IsA("NumberValue") then return stun.Value > 0 end
        return true
    end
    local rag = char:GetAttribute("Ragdoll")
    return rag ~= nil and rag ~= 0 and rag ~= false
end

local function targetM1(target)
    return target and target:GetAttribute("CurrentM1") or nil
end

local function readFlag(target, name)
    if not target then return false end
    local info = target:FindFirstChild("Info")
    local obj = info and info:FindFirstChild(name)
    if obj and obj:IsA("BoolValue") then return obj.Value end
    if obj and obj:IsA("NumberValue") then return obj.Value ~= 0 end
    local attr = target:GetAttribute(name)
    if typeof(attr) == "boolean" then return attr end
    if typeof(attr) == "number" then return attr ~= 0 end
    return false
end

local function targetIsAttacking(target)
    return readFlag(target, "InSkill") or readFlag(target, "NoJump") or readFlag(target, "NoSprint")
end

-- TWO M1 + HABILIDADES 1-4 AUTOMATICAS
-- Tenta usar cada habilidade quando o cooldown local informado estiver pronto.
local function pressAbilityKey(keyCode)
    local pressed = pcall(function()
        VirtualInputManager:SendKeyEvent(true, keyCode, false, game)
    end)
    if not pressed then return false end
    task.wait(ABILITY_KEY_HOLD)
    pcall(function()
        VirtualInputManager:SendKeyEvent(false, keyCode, false, game)
    end)
    return true
end

local function useReadyAbilities(target)
    for _, ability in ipairs(ABILITIES) do
        if not Running or Recovering or not targetAlive(target) then return end
        local now = os.clock()
        local lastUse = LastAbilityUse[ability.name] or -math.huge
        if now - lastUse >= ability.cooldown then
            status.Text = "HABILIDADE: " .. ability.name .. "\nTecla " .. tostring(ability.key.Name)
            if pressAbilityKey(ability.key) then
                LastAbilityUse[ability.name] = os.clock()
                task.wait(ABILITY_GAP)
            else
                status.Text = "Não foi possível simular a tecla " .. tostring(ability.key.Name)
                return
            end
        end
    end
end

local function doFourM1(target)
    if BusyCombo or Recovering or Blocking or FullM1Defense or FourthOnlyDefense then return end
    BusyCombo = true

    -- Faz somente dois M1 antes de tentar as habilidades.
    for i = 1, 2 do
        if not Running or Recovering or not targetAlive(target) then break end
        if healthPercent() <= LOW_HP_PERCENT then break end
        if FullM1Defense or FourthOnlyDefense or Blocking then break end
        while Running and not Recovering and targetAlive(target) and distanceTo(target) > ATTACK_DISTANCE do
            updateTweenToTarget(target)
            if FullM1Defense or FourthOnlyDefense or healthPercent() <= LOW_HP_PERCENT then break end
            task.wait(0.02)
        end
        if not Running or Recovering or FullM1Defense or FourthOnlyDefense or not targetAlive(target) then break end
        stopTween()
        while Running and not Recovering and amStunned() do task.wait(0.03) end
        if not Running or Recovering or not targetAlive(target) then break end
        faceTarget(target)
        status.Text = "ATAQUE\nM1 " .. i .. " / 2"
        if not oneM1(target) then break end
        task.wait(M1_DELAY)
    end

    if Running and not Recovering and targetAlive(target)
        and not FullM1Defense and not FourthOnlyDefense and not Blocking
        and healthPercent() > LOW_HP_PERCENT then
        useReadyAbilities(target)
    end
    BusyCombo = false
end

-- DRINK / SHOP
local function nameMatches(text, list)
    text = string.lower(tostring(text))
    for _, word in ipairs(list) do
        if string.find(text, word, 1, true) then return true end
    end
    return false
end

local function getObjectPosition(obj)
    if obj:IsA("BasePart") then return obj.Position end
    if obj:IsA("Model") then
        local ok, pivot = pcall(function() return obj:GetPivot() end)
        if ok then return pivot.Position end
    end
    local part = obj:FindFirstAncestorWhichIsA("BasePart")
    if part then return part.Position end
    local model = obj:FindFirstAncestorWhichIsA("Model")
    if model then
        local ok, pivot = pcall(function() return model:GetPivot() end)
        if ok then return pivot.Position end
    end
    return nil
end

local function findDrinkShop()
    local best, bestPos, bestHeight = nil, nil, -math.huge
    for _, obj in ipairs(workspace:GetDescendants()) do
        if nameMatches(obj.Name, DRINK_NAMES) then
            local hasInteraction = obj:IsA("ProximityPrompt") or obj:IsA("ClickDetector")
                or obj:FindFirstChildWhichIsA("ProximityPrompt", true)
                or obj:FindFirstChildWhichIsA("ClickDetector", true)
            if hasInteraction then
                local pos = getObjectPosition(obj)
                if pos and pos.Y > bestHeight then bestHeight, best, bestPos = pos.Y, obj, pos end
            end
        end
    end
    return best, bestPos
end

local function interactWithShop(obj)
    if not obj then return end
    local prompts, clicks = {}, {}
    if obj:IsA("ProximityPrompt") then table.insert(prompts, obj) end
    if obj:IsA("ClickDetector") then table.insert(clicks, obj) end
    for _, d in ipairs(obj:GetDescendants()) do
        if d:IsA("ProximityPrompt") then table.insert(prompts, d)
        elseif d:IsA("ClickDetector") then table.insert(clicks, d) end
    end
    for _ = 1, SHOP_INTERACT_TRIES do
        if not Running or not Recovering then break end
        for _, prompt in ipairs(prompts) do
            if fireproximityprompt then pcall(fireproximityprompt, prompt) end
        end
        for _, click in ipairs(clicks) do
            if fireclickdetector then pcall(fireclickdetector, click) end
        end
        task.wait(0.18)
    end
end

local function findDrinkTool()
    local backpack = LP:FindFirstChildOfClass("Backpack")
    local char = LP.Character
    for _, location in ipairs({backpack, char}) do
        if location then
            for _, obj in ipairs(location:GetChildren()) do
                if obj:IsA("Tool") and nameMatches(obj.Name, TOOL_DRINK_NAMES) then return obj end
            end
        end
    end
    return nil
end

local function useDrink()
    local char, hum = getMyCharacter()
    if not char or not hum then return false end
    local tool = findDrinkTool()
    if not tool then return false end
    local backpack = LP:FindFirstChildOfClass("Backpack")
    if tool.Parent == backpack then hum:EquipTool(tool); task.wait(0.15) end
    pcall(function() tool:Activate() end)
    task.wait(0.35)
    return true
end

-- RECOVERY: unico estado; nao retorna ao combate ate terminar a rotina.
local function recoverHealth()
    if Recovering then return end
    Recovering = true

    -- Interrompe o combate e a defesa antes de subir para recuperar vida.
    stopTween()
    stopBlock()
    stopCombatFacing()
    BusyCombo = false
    CurrentTarget = nil
    if disconnectTargetMonitor then disconnectTargetMonitor() end

    RecoveryGeneration += 1
    local myGeneration = RecoveryGeneration
    FullM1Defense = false
    FourthOnlyDefense = false
    DefenseGeneration += 1

    local character, hum, root = getMyCharacter()
    if not character or not hum or not root then
        restoreNoclip()
        Recovering = false
        return
    end

    startNoclip()
    local hoverPosition = root.Position + Vector3.new(0, 100, 0)
    status.Text = "HP BAIXO\\nSUBINDO PARA RECUPERAR..."

    -- Sobe verticalmente e depois mantém a posição elevada enquanto recupera.
    local arrived = tweenLeg(root, hoverPosition, SHOP_TWEEN_SPEED)
    if not arrived or not Running or not Recovering or myGeneration ~= RecoveryGeneration
        or LP.Character ~= character or not hum.Parent or hum.Health <= 0 or not root.Parent then
        restoreNoclip()
        Recovering = false
        status.Text = "Recuperação interrompida: personagem mudou ou morreu."
        return
    end

    local hoverConnection
    hoverConnection = RunService.Heartbeat:Connect(function()
        if not Running or not Recovering or myGeneration ~= RecoveryGeneration
            or LP.Character ~= character or not hum.Parent or hum.Health <= 0 or not root.Parent then
            if hoverConnection then hoverConnection:Disconnect() end
            return
        end
        root.CFrame = CFrame.new(hoverPosition, hoverPosition + root.CFrame.LookVector)
        root.AssemblyLinearVelocity = Vector3.zero
        root.AssemblyAngularVelocity = Vector3.zero
    end)

    local deadline = os.clock() + RECOVERY_WAIT_LIMIT
    while Running and Recovering and myGeneration == RecoveryGeneration and os.clock() < deadline do
        -- Não fica esperando o timeout inteiro se morrer ou ocorrer respawn no meio da recuperação.
        if LP.Character ~= character or not hum.Parent or hum.Health <= 0 or not root.Parent then
            break
        end
        updateHP()
        local hp = healthPercent()
        status.Text = "RECUPERANDO NO ALTO\\n" .. math.floor(hp * 100) .. "% / "
            .. math.floor(RECOVER_TO_PERCENT * 100) .. "%"
        if hp >= RECOVER_TO_PERCENT then break end
        task.wait(0.2)
    end

    if hoverConnection then hoverConnection:Disconnect() end
    if myGeneration ~= RecoveryGeneration then
        restoreNoclip()
        return
    end

    restoreNoclip()
    Recovering = false
    CurrentTarget = nil
    if healthPercent() >= RECOVER_TO_PERCENT then
        status.Text = "RECUPERAÇÃO FINALIZADA\\nVoltando ao combate"
    else
        status.Text = "HP não recuperou a tempo.\\nTentará novamente se necessário."
    end
    task.wait(0.1)
end

-- TARGET MONITOR / DEFENSE
disconnectTargetMonitor = function()
    for _, c in ipairs(TargetConnections) do pcall(function() c:Disconnect() end) end
    table.clear(TargetConnections)
    DefenseGeneration += 1
    FullM1Defense = false
    FourthOnlyDefense = false
    LastTargetM1 = nil
    LastM1ChangeTime = 0
end

local function clearM1Defense(target, generation)
    if generation ~= DefenseGeneration or target ~= CurrentTarget then return end
    FullM1Defense = false
    FourthOnlyDefense = false
    stopBlock()
    status.Text = "DEFESA LIBERADA\nCombo interrompido/terminado"
end

local function armDefenseTimeout(target)
    DefenseGeneration += 1
    local generation = DefenseGeneration
    LastM1ChangeTime = os.clock()
    -- Polling guard: always releases after the deadline, even if the target's
    -- CurrentM1 attribute remains stuck at 1 or 4 and never changes again.
    task.spawn(function()
        while Running and not Recovering and target == CurrentTarget
            and generation == DefenseGeneration do
            if os.clock() - LastM1ChangeTime >= M1_DEFENSE_TIMEOUT then
                clearM1Defense(target, generation)
                return
            end
            task.wait(0.05)
        end
    end)
end

local function monitorTarget(target)
    disconnectTargetMonitor()
    if not target then return end
    LastTargetM1 = target:GetAttribute("CurrentM1")
    table.insert(TargetConnections, target:GetAttributeChangedSignal("CurrentM1"):Connect(function()
        if not Running or Recovering or target ~= CurrentTarget then return end
        local oldM1 = LastTargetM1
        local newM1 = target:GetAttribute("CurrentM1")
        LastTargetM1 = newM1

        if typeof(newM1) == "number" and newM1 >= 1 and newM1 <= 4 then
            if newM1 == 1 then
                FullM1Defense = true
                FourthOnlyDefense = false
                BusyCombo = false
                status.Text = "M1 1 DETECTADO\nDEFENDENDO ATÉ O FIM"
                if distanceTo(target) <= BLOCK_DISTANCE then startBlock(target) end
            elseif newM1 == 4 then
                if not FullM1Defense then FourthOnlyDefense = true end
                BusyCombo = false
                status.Text = "M1 4 DETECTADO\nDEFENDENDO"
                if distanceTo(target) <= BLOCK_DISTANCE then startBlock(target) end
            end
            armDefenseTimeout(target)
        elseif oldM1 ~= newM1 then
            DefenseGeneration += 1
            FullM1Defense = false
            FourthOnlyDefense = false
            stopBlock()
            status.Text = "M1 TERMINOU\nDEFESA LIBERADA"
        end
    end))
end

local function setTarget(target)
    if Recovering then return end
    if CurrentTarget == target then return end
    stopTween()
    stopBlock()
    disconnectTargetMonitor()
    CurrentTarget = target
    if target then
        monitorTarget(target)
        startCombatFacing()
    else
        stopCombatFacing()
    end
end

-- FIGHT
local function fightTarget(target)
    if Recovering then return end
    if healthPercent() <= LOW_HP_PERCENT then recoverHealth(); return end
    if not targetAlive(target) then return end
    local m1 = targetM1(target)
    local dist = distanceTo(target)
    targetLabel.Text = "Alvo: " .. target.Name .. "\nTipo: " .. getTargetType(target)
        .. " | M1: " .. tostring(m1 or "?") .. "\nDist: " .. string.format("%.1f", dist)

    if FullM1Defense or FourthOnlyDefense then
        status.Text = (FullM1Defense and "DEFESA M1 ATIVA" or "DEFESA DO M1 4") .. "\nM1 atual: " .. tostring(m1)
        if dist <= BLOCK_DISTANCE then startBlock(target) else updateTweenToTarget(target) end
        return
    end

    if m1 == 4 and targetIsAttacking(target) then
        FourthOnlyDefense = true
        BusyCombo = false
        armDefenseTimeout(target)
        startBlock(target)
        return
    end

    if dist > ATTACK_DISTANCE then
        status.Text = "PERSEGUINDO\nAcompanhando alvo..."
        updateTweenToTarget(target)
        return
    end
    stopTween()
    faceTarget(target)
    if not BusyCombo and not Blocking and not amStunned() then doFourM1(target) end
end

-- FARM LOOP
local function farmLoop()
    while Running do
        updateHP()
        if tonumber(KillsValue.Value) and tonumber(KillsValue.Value) >= GOAL then Running = false; break end
        local char = getMyCharacter()
        if not char then
            stopTween(); stopBlock()
            status.Text = "Esperando respawn..."
            task.wait(0.2)
            continue
        end
        if healthPercent() <= LOW_HP_PERCENT and not Recovering then recoverHealth(); continue end
        if Recovering then task.wait(0.05); continue end

        if not CurrentTarget or not targetAlive(CurrentTarget) then
            local target = findNearestTarget()
            if not target then
                setTarget(nil)
                targetLabel.Text = "Alvo: nenhum"
                status.Text = "Esperando alvo..."
                task.wait(0.15)
                continue
            end
            setTarget(target)
        end

        fightTarget(CurrentTarget)
        if CurrentTarget and not targetAlive(CurrentTarget) then
            setTarget(nil)
            status.Text = "Alvo derrotado\nProcurando próximo..."
            task.wait(0.08)
        end
        task.wait(LOOP_DELAY)
    end

    stopTween(); stopBlock(); stopCombatFacing(); disconnectTargetMonitor()
    if tonumber(KillsValue.Value) and tonumber(KillsValue.Value) >= GOAL then
        status.Text = "META CONCLUÍDA ✓"
        toggle.Text = "META CONCLUÍDA"
    else
        status.Text = "Status: parado"
        toggle.Text = "INICIAR"
    end
end

-- Stop hook shared with future executions: prevents old farm/recovery loops fighting the new copy.
ScriptGeneration += 1
local thisScriptGeneration = ScriptGeneration
SharedEnv.GojoSmartKaitunStop = function()
    Running = false
    Recovering = false
    restoreNoclip()
    restoreMapVisuals()
    if NoclipCharacterConnection then
        NoclipCharacterConnection:Disconnect()
        NoclipCharacterConnection = nil
    end
    RecoveryGeneration += 1
    stopTween()
    stopBlock()
    stopCombatFacing()
    disconnectTargetMonitor()
    if gui and gui.Parent then gui:Destroy() end
end

-- KILLS / TOGGLE
updateKills()
KillsValue.Changed:Connect(function()
    updateKills()
    if tonumber(KillsValue.Value) and tonumber(KillsValue.Value) >= GOAL then Running = false end
end)

toggle.MouseButton1Click:Connect(function()
    if tonumber(KillsValue.Value) and tonumber(KillsValue.Value) >= GOAL then status.Text = "Meta já concluída."; return end
    Running = not Running
    if Running then
        toggle.Text = "PARAR"
        status.Text = "Iniciando..."
        task.spawn(farmLoop)
    else
        stopTween(); stopBlock(); stopCombatFacing(); disconnectTargetMonitor()
        CurrentTarget = nil
        BusyCombo = false
        Recovering = false
        RecoveryGeneration += 1
        restoreNoclip()
        FullM1Defense = false
        FourthOnlyDefense = false
        targetLabel.Text = "Alvo: nenhum"
        status.Text = "Status: parado"
        toggle.Text = "INICIAR"
    end
end)

-- MOVABLE UI
local dragging, dragStart, startPos, dragInput = false, nil, nil, nil
title.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
        dragging = true
        dragStart = input.Position
        startPos = frame.Position
        input.Changed:Connect(function() if input.UserInputState == Enum.UserInputState.End then dragging = false end end)
    end
end)
title.InputChanged:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseMovement then dragInput = input end
end)
UIS.InputChanged:Connect(function(input)
    if dragging and input == dragInput then
        local delta = input.Position - dragStart
        frame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
    end
end)

close.MouseButton1Click:Connect(function()
    Running = false
    Recovering = false
    RecoveryGeneration += 1
    restoreNoclip()
    restoreMapVisuals()
    if NoclipCharacterConnection then
        NoclipCharacterConnection:Disconnect()
        NoclipCharacterConnection = nil
    end
    stopTween(); stopBlock(); stopCombatFacing(); disconnectTargetMonitor()
    gui:Destroy()
end)

updateHP()
print("GOJO SMART KAITUN V8 + 2 M1 e habilidades automáticas carregado |", KillsValue.Value, "/", GOAL)
