-- FX Hub v4.6 COMPLETO - FLY GUI BOTÃO (executa QUANTAS VEZES QUISER)
local Sound = Instance.new("Sound")
Sound.SoundId = "rbxassetid://8449305114"
Sound.Volume = 4
Sound.Parent = game:GetService("SoundService")
Sound:Play()

game:GetService("StarterGui"):SetCore("SendNotification", {
    Title = "FX Hub Rayfield",
    Text = "Script ativado com sucesso!",
    Duration = 5,
})

local Rayfield = loadstring(game:HttpGet('https://sirius.menu/rayfield'))()

local Window = Rayfield:CreateWindow({
    Name = "FX Hub v4.6",
    LoadingTitle = "Carregando FX Hub...",
    LoadingSubtitle = "ESP Linha/Name separados + ESP persiste + TODOS PLAYERS + FLY GUI",
    ConfigurationSaving = {
        Enabled = true,
        FolderName = "FXHub",
        FileName = "Config"
    }
})

local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
local Lighting = game:GetService("Lighting")
local camera = Workspace.CurrentCamera

local aimbotEnabled = false
local aimbotTargetPart = "Head"
local killCheckEnabled = true
local wallCheckEnabled = true
local fovSize = 120
local smoothPower = 0.35
local AimbotTeamCheck = true
local fovColor = Color3.fromRGB(170, 0, 255)
local config = {
    killCheckRange = 150,
    defaultDamageEstimate = 9999,
    useCameraForRay = true,
    raycastFilterType = Enum.RaycastFilterType.Blacklist,
    raycastBlacklist = {}
}

local ESPLineEnabled = false
local ESPNameEnabled = false
local ESPTeamCheck = false
local ESPObjects = {}

-- FLY GUI - EXECUTA SEMPRE QUE CLICAR (SEM LIMITAÇÃO)
local function loadFlyGUI()
    Rayfield:Notify({
        Title = "Fly GUI",
        Content = "Carregando...",
        Duration = 1.5
    })
    
    pcall(function()
        loadstring(game:HttpGet("https://rawscripts.net/raw/Universal-Script-fly-Gui-v5-64340"))()
        Rayfield:Notify({
            Title = "Fly GUI",
            Content = "✅ Executado!",
            Duration = 2
        })
    end)
end

-- [TODAS AS FUNÇÕES ESP/AIMBOT IGUAIS - SEM MUDANÇAS]
local function getPlayerRankColor(player)
    if not player or not player.Character then return Color3.new(1,0,0) end
    local leaderstats = player:FindFirstChild("leaderstats")
    if leaderstats then
        for _, stat in pairs(leaderstats:GetChildren()) do
            if stat:IsA("StringValue") or stat:IsA("IntValue") or stat:IsA("NumberValue") then
                local colorValue = leaderstats:FindFirstChild(stat.Name.."Color") or player:FindFirstChild(stat.Name.."Color")
                if colorValue and colorValue:IsA("Color3Value") then
                    return colorValue.Value
                end
            end
        end
    end
    if player.Team and player.TeamColor then
        return player.TeamColor.Color
    end
    if player:GetAttribute("RankColor") then return player:GetAttribute("RankColor") end
    if player.Character and player.Character:GetAttribute("TeamColor") then
        return player.Character:GetAttribute("TeamColor")
    end
    local playerGui = player:FindFirstChild("PlayerGui")
    if playerGui then
        for _, gui in pairs(playerGui:GetDescendants()) do
            if (gui:IsA("TextLabel") or gui:IsA("TextButton")) and gui.TextColor3 then
                if gui.Parent.Name:lower():find("rank") or gui.Parent.Name:lower():find("class") then
                    return gui.TextColor3
                end
            end
        end
    end
    return Color3.new(1,0,0)
end

local function refreshRaycastBlacklist()
    config.raycastBlacklist = {}
    if LocalPlayer.Character then table.insert(config.raycastBlacklist, LocalPlayer.Character) end
    for _, v in pairs(LocalPlayer:GetDescendants()) do
        if v:IsA("Tool") then table.insert(config.raycastBlacklist, v) end
    end
end
LocalPlayer.CharacterAdded:Connect(refreshRaycastBlacklist)
refreshRaycastBlacklist()

local function isEnemy(player)
    if not player or player == LocalPlayer then return false end
    if not AimbotTeamCheck then return true end
    if player.Team and LocalPlayer.Team then
        return player.Team ~= LocalPlayer.Team
    end
    return true
end

local function isVisible(targetPart)
    if not targetPart or not targetPart.Parent then return false end
    local origin = camera.CFrame.Position
    local direction = targetPart.Position - origin
    local rayParams = RaycastParams.new()
    rayParams.FilterType = config.raycastFilterType
    rayParams.FilterDescendantsInstances = config.raycastBlacklist
    rayParams.IgnoreWater = true
    local result = Workspace:Raycast(origin, direction, rayParams)
    if not result then return true end
    return result.Instance:IsDescendantOf(targetPart.Parent)
end

local function isKillable(player, damageEstimate)
    local hum = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
    if not hum or hum.Health <= 0 then return false end
    if not isEnemy(player) then return false end
    if not player.Character:FindFirstChild(aimbotTargetPart) then return false end
    if (player.Character[aimbotTargetPart].Position - camera.CFrame.Position).Magnitude > config.killCheckRange then return false end
    local estimate = damageEstimate or config.defaultDamageEstimate
    return hum.Health <= estimate
end

local function getClosestTarget()
    local shortestDist = math.huge
    local closest = nil
    local screenCenter = Vector2.new(camera.ViewportSize.X / 2, camera.ViewportSize.Y / 2)
    for _, player in pairs(Players:GetPlayers()) do
        if isEnemy(player) and player.Character then
            local targetPart = player.Character:FindFirstChild(aimbotTargetPart)
            if targetPart then
                local screenPos, onScreen = camera:WorldToViewportPoint(targetPart.Position)
                if onScreen then
                    local dist = (Vector2.new(screenPos.X, screenPos.Y) - screenCenter).Magnitude
                    if dist < fovSize and dist < shortestDist then
                        if (not killCheckEnabled or isKillable(player)) and (not wallCheckEnabled or isVisible(targetPart)) then
                            closest = targetPart
                            shortestDist = dist
                        end
                    end
                end
            end
        end
    end
    return closest
end

local Drawing = (pcall(function() return Drawing end) and Drawing) or nil
local FovCircle
if Drawing then
    FovCircle = Drawing.new("Circle")
    FovCircle.Visible = false
    FovCircle.Filled = false
    FovCircle.Color = fovColor
    FovCircle.Thickness = 2
    FovCircle.Radius = fovSize
    FovCircle.Position = Vector2.new(camera.ViewportSize.X / 2, camera.ViewportSize.Y / 2)
end

local function createESP(player)
    if ESPObjects[player] then return end
    local box = Drawing.new("Square")
    box.Thickness = 2
    box.Filled = false
    box.Transparency = 1
    box.Color = Color3.new(1,0,0)
    box.Visible = false

    local nameTag = Drawing.new("Text")
    nameTag.Size = 14
    nameTag.Center = true
    nameTag.Outline = true
    nameTag.Font = 2
    nameTag.Color = Color3.new(1,1,1)
    nameTag.Visible = false

    local tracerLine = Drawing.new("Line")
    tracerLine.Thickness = 4
    tracerLine.Color = Color3.new(1,0,0)
    tracerLine.Transparency = 0.6
    tracerLine.Visible = false

    ESPObjects[player] = {box = box, name = nameTag, tracer = tracerLine}
end

local function removeESP(player)
    if ESPObjects[player] then
        local esp = ESPObjects[player]
        if esp.box then esp.box:Remove() end
        if esp.name then esp.name:Remove() end
        if esp.tracer then esp.tracer:Remove() end
        ESPObjects[player] = nil
    end
end

local function shouldShowESP(player)
    if not player or player == LocalPlayer then return false end
    if not ESPTeamCheck then return true end
    return player.Team ~= LocalPlayer.Team
end

local function updateESP()
    if not Drawing then return end
    local screenCenter = Vector2.new(camera.ViewportSize.X / 2, camera.ViewportSize.Y * 0.9)

    for player, esp in pairs(ESPObjects) do
        if not player or not player.Parent or not player.Character or not player.Character:FindFirstChild("HumanoidRootPart") then
            esp.name.Visible = false
            esp.box.Visible = false
            esp.tracer.Visible = false
        elseif shouldShowESP(player) then
            local rootPart = player.Character.HumanoidRootPart
            local headPos = rootPart.Position + Vector3.new(0,3,0)
            local footPos = rootPart.Position - Vector3.new(0,4,0)

            local top, onScreenTop = camera:WorldToViewportPoint(headPos)
            local bottom, onScreenBottom = camera:WorldToViewportPoint(footPos)

            local chestPart = player.Character:FindFirstChild("UpperTorso")
                            or player.Character:FindFirstChild("Torso")
                            or rootPart
            local chestScreen, onScreenChest = camera:WorldToViewportPoint(chestPart.Position)

            if onScreenTop or onScreenChest or onScreenBottom then
                local rankColor = getPlayerRankColor(player)

                if ESPNameEnabled then
                    local dist = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") and math.floor((LocalPlayer.Character.HumanoidRootPart.Position - rootPart.Position).Magnitude) or 0
                    esp.name.Text = player.Name .. " [" .. dist .. "]"
                    esp.name.Position = Vector2.new(top.X, top.Y - 20)
                    esp.name.Color = rankColor
                    esp.name.Visible = true
                else
                    esp.name.Visible = false
                end

                esp.box.Visible = false

                if ESPLineEnabled then
                    if onScreenChest then
                        esp.tracer.From = screenCenter
                        esp.tracer.To = Vector2.new(chestScreen.X, chestScreen.Y)
                        esp.tracer.Color = rankColor
                        esp.tracer.Visible = true
                    else
                        esp.tracer.Visible = false
                    end
                else
                    esp.tracer.Visible = false
                end
            else
                esp.name.Visible = false
                esp.box.Visible = false
                esp.tracer.Visible = false
            end
        else
            esp.name.Visible = false
            esp.box.Visible = false
            esp.tracer.Visible = false
        end
    end

    for _, player in pairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and not ESPObjects[player] and player.Character then
            createESP(player)
        end
    end
end

local function toggleESPLine(state)
    ESPLineEnabled = state
    if not state then
        for _, esp in pairs(ESPObjects) do
            if esp.tracer then esp.tracer.Visible = false end
        end
    else
        for _, player in pairs(Players:GetPlayers()) do
            if player ~= LocalPlayer then
                createESP(player)
            end
        end
    end
end

local function toggleESPName(state)
    ESPNameEnabled = state
    if not state then
        for _, esp in pairs(ESPObjects) do
            if esp.name then esp.name.Visible = false end
        end
    else
        for _, player in pairs(Players:GetPlayers()) do
            if player ~= LocalPlayer then
                createESP(player)
            end
        end
    end
end

for _, player in pairs(Players:GetPlayers()) do
    if player ~= LocalPlayer then
        if player.Character then
            createESP(player)
        end
        player.CharacterAdded:Connect(function()
            wait(0.1)
            createESP(player)
        end)
    end
end

Players.PlayerAdded:Connect(function(player)
    if player ~= LocalPlayer then
        player.CharacterAdded:Connect(function()
            wait(0.1)
            createESP(player)
        end)
    end
end)

Players.PlayerRemoving:Connect(removeESP)

local NoclipConnection = nil
local function toggleNoclip(state)
    if NoclipConnection then
        NoclipConnection:Disconnect()
        NoclipConnection = nil
    end
    if state then
        NoclipConnection = RunService.Stepped:Connect(function()
            if LocalPlayer.Character then
                for _, part in pairs(LocalPlayer.Character:GetChildren()) do
                    if part:IsA("BasePart") and part.Name ~= "HumanoidRootPart" then
                        part.CanCollide = false
                    end
                end
            end
        end)
    end
end

local MovementTab = Window:CreateTab("Movimentação", 4483362458)
local VisualTab = Window:CreateTab("Visual", 4483362458)
local PvPTab = Window:CreateTab("PvP", 4483362458)

MovementTab:CreateSlider({
    Name = "WalkSpeed",
    Range = {16, 250},
    Increment = 1,
    Suffix = "Speed",
    CurrentValue = 16,
    Flag = "WalkSpeed",
    Callback = function(value)
        if LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Humanoid") then
            LocalPlayer.Character:FindFirstChildOfClass("Humanoid").WalkSpeed = value
        end
    end
})

MovementTab:CreateToggle({
    Name = "Noclip",
    CurrentValue = false,
    Flag = "Noclip",
    Callback = function(state)
        toggleNoclip(state)
    end
})

-- BOTÃO FLY GUI (QUANTAS VEZES QUISER)
MovementTab:CreateButton({
    Name = "Fly GUI",
    Callback = function()
        loadFlyGUI()
    end
})

VisualTab:CreateToggle({
    Name = "ESP Linha",
    CurrentValue = false,
    Flag = "ESPLine",
    Callback = function(state)
        toggleESPLine(state)
    end
})

VisualTab:CreateToggle({
    Name = "ESP Name",
    CurrentValue = false,
    Flag = "ESPName",
    Callback = function(state)
        toggleESPName(state)
    end
})

VisualTab:CreateToggle({
    Name = "Team Check (só inimigos)",
    CurrentValue = false,
    Flag = "ESPTeamCheck",
    Callback = function(state)
        ESPTeamCheck = state
    end
})

VisualTab:CreateToggle({
    Name = "Antilag",
    CurrentValue = false,
    Flag = "Antilag",
    Callback = function(state)
        if state then
            Lighting.GlobalShadows = false
            for _, v in pairs(Lighting:GetChildren()) do
                if v:IsA("PostEffect") or v:IsA("Sky") then
                    v.Enabled = false
                end
            end
            settings().Rendering.QualityLevel = Enum.QualityLevel.Level01
        else
            Lighting.GlobalShadows = true
            for _, v in pairs(Lighting:GetChildren()) do
                if v:IsA("PostEffect") then
                    v.Enabled = true
                end
            end
            settings().Rendering.QualityLevel = Enum.QualityLevel.Automatic
        end
    end
})

PvPTab:CreateToggle({
    Name = "Aimbot",
    CurrentValue = false,
    Flag = "Aimbot",
    Callback = function(state)
        aimbotEnabled = state
        if Drawing and FovCircle then
            FovCircle.Visible = state
        end
    end
})

PvPTab:CreateDropdown({
    Name = "Parte do Corpo",
    Options = {"Head", "Torso"},
    CurrentOption = {"Head"},
    Flag = "AimbotPart",
    Callback = function(option)
        aimbotTargetPart = option[1]
        Rayfield:Notify({
            Title = "Aimbot",
            Content = "Mirando em: " .. aimbotTargetPart,
            Duration = 2,
        })
    end
})

PvPTab:CreateSlider({
    Name = "FOV",
    Range = {50,500},
    Increment = 10,
    Suffix = "px",
    CurrentValue = 120,
    Flag = "FOV",
    Callback = function(value)
        fovSize = value
        if Drawing and FovCircle then
            FovCircle.Radius = value
        end
    end
})

PvPTab:CreateColorPicker({
    Name = "Cor FOV",
    Color = Color3.fromRGB(170,0,255),
    Flag = "FOVColor",
    Callback = function(color)
        fovColor = color
        if Drawing and FovCircle then
            FovCircle.Color = color
        end
    end,
})

PvPTab:CreateToggle({
    Name = "Team Check (só inimigos)",
    CurrentValue = true,
    Flag = "AimbotTeamCheck",
    Callback = function(state)
        AimbotTeamCheck = state
    end
})

PvPTab:CreateToggle({
    Name = "KillCheck",
    CurrentValue = true,
    Flag = "KillCheck",
    Callback = function(state)
        killCheckEnabled = state
    end
})

PvPTab:CreateToggle({
    Name = "WallCheck",
    CurrentValue = true,
    Flag = "WallCheck",
    Callback = function(state)
        wallCheckEnabled = state
    end
})

local aimbotForca = "Forte"
local aimbotSmoothMap = {
    ["Fraco"] = 0.15,
    ["Médio"] = 0.25,
    ["Forte"] = 0.35,
    ["Insano"] = 0.6,
    ["Hacker"] = 0.85
}

PvPTab:CreateDropdown({
    Name = "Força",
    Options = {"Fraco","Médio","Forte","Insano","Hacker"},
    CurrentOption = {"Forte"},
    Flag = "ForcaAimbot",
    Callback = function(option)
        aimbotForca = option[1]
        smoothPower = aimbotSmoothMap[aimbotForca]
    end
})

RunService.RenderStepped:Connect(function()
    if aimbotEnabled and camera.CFrame then
        local target = getClosestTarget()
        if target then
            camera.CFrame = camera.CFrame:lerp(CFrame.new(camera.CFrame.Position,target.Position), smoothPower)
        end
    end
    
    if Drawing and FovCircle then
        FovCircle.Position = Vector2.new(camera.ViewportSize.X/2,camera.ViewportSize.Y/2)
        FovCircle.Color = fovColor
    end
    
    updateESP()
end)

Rayfield:Notify({
    Title = "FX Hub v4.6",
    Content = "✅ ESP TODOS players + Fly GUI (INFINITAS VEZES)",
    Duration = 6,
    Image = 4483362458
})

print("[FX Hub v4.6] Script carregado! Fly GUI = BOTÃO INFINITO!")-- ADICIONE ISSO ANTES das outras abas (MovementTab, VisualTab, PvPTab)

local TeleportService = game:GetService("TeleportService")
local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer

local serverJobId = game.JobId
local placeId = game.PlaceId
local teleportIdInput = ""

local function copyServerId()
    if setclipboard then
        setclipboard(serverJobId)
        Rayfield:Notify({
            Title = "Servidor",
            Content = "ID copiado: " .. string.sub(serverJobId, 1, 20) .. "...",
            Duration = 4,
        })
    else
        Rayfield:Notify({
            Title = "Erro",
            Content = "Clipboard não suportado. ID: " .. serverJobId,
            Duration = 5,
        })
    end
end

local function teleportToServerWithId(id)
    if not id or id == "" or id == serverJobId then
        Rayfield:Notify({
            Title = "Erro",
            Content = "ID inválido ou é o servidor atual!",
            Duration = 3,
        })
        return
    end
    
    Rayfield:Notify({
        Title = "Teleportando...",
        Content = "Para: " .. string.sub(id, 1, 20) .. "...",
        Duration = 3,
    })
    
    local success, errorMessage = pcall(function()
        TeleportService:TeleportToPlaceInstance(placeId, id, LocalPlayer)
    end)
    
    if not success then
        Rayfield:Notify({
            Title = "Erro no Teleport",
            Content = tostring(errorMessage),
            Duration = 5,
        })
    end
end

-- CRIA A ABA SERVIDOR (adicione ANTES das outras abas)
local ServerTab = Window:CreateTab("Servidor", 4483362458)

ServerTab:CreateButton({
    Name = "Copiar ID Servidor",
    Callback = copyServerId
})

ServerTab:CreateInput({
    Name = "ID Servidor (Cole aqui)",
    PlaceholderText = "ex: 1234567890_1234567890_1234567890",
    RemoveTextAfterFocusLost = false,
    Callback = function(text)
        teleportIdInput = text
    end
})

ServerTab:CreateButton({
    Name = "Teleportar",
    Callback = function()
        teleportToServerWithId(teleportIdInput)
    end
})

-- AGORA continue com suas outras abas (MovementTab, VisualTab, PvPTab) normalmente-- Função Infinite Jump
local UserInputService = game:GetService("UserInputService")
local infiniteJumpEnabled = false

UserInputService.JumpRequest:Connect(function()
    if infiniteJumpEnabled then
        if LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Humanoid") then
            LocalPlayer.Character:FindFirstChildOfClass("Humanoid"):ChangeState(Enum.HumanoidStateType.Jumping)
        end
    end
end)

MovementTab:CreateToggle({
    Name = "Infinite Jump",
    CurrentValue = false,
    Flag = "InfiniteJump",
    Callback = function(state)
        infiniteJumpEnabled = state
        Rayfield:Notify({
            Title = "Infinite Jump",
            Content = state and "Ativado" or "Desativado",
            Duration = 2,
        })
    end,
})
