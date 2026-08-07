-- ==========================================================
-- PANEL SQUEEZE - SOUTH BRONX (OCEAN THEME)
-- ==========================================================

-- ----------------------------------------------------------
-- 🛡️ SISTEMA DE PROTECCIÓN ANTI-BAN & BYPASS DE METATABLES
-- ----------------------------------------------------------
local getrawmetatable = getrawmetatable or debug.getmetatable
local setreadonly = setreadonly or make_writeable or setmetatablereadonly

if getrawmetatable and setreadonly then
    local RawMetatable = getrawmetatable(game)
    setreadonly(RawMetatable, false)
    
    local oldNamecall = RawMetatable.__namecall
    local oldIndex = RawMetatable.__index

    RawMetatable.__namecall = newcclosure(function(self, ...)
        local method = getnamecallmethod()
        if method == "Kick" or method == "kick" then
            return nil
        end
        return oldNamecall(self, ...)
    end)

    RawMetatable.__index = newcclosure(function(self, key)
        if not checkcaller() and self:IsA("Humanoid") and key == "WalkSpeed" then
            return 16
        end
        return oldIndex(self, key)
    end)
    
    setreadonly(RawMetatable, true)
end

-- ----------------------------------------------------------
-- SCRIPT PRINCIPAL
-- ----------------------------------------------------------
local Rayfield = loadstring(game:HttpGet('https://sirius.menu/rayfield'))()

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")

local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera

-- ==========================================================
-- CONFIGURACIÓN GLOBAL
-- ==========================================================
local Settings = {
    AimbotEnabled = false,
    AimMode = "Camera",
    Activation = "Hold",
    Keybind = Enum.KeyCode.E,
    FOV = 100,
    Smoothness = 3,
    ShowFOVCircle = false,
    RainbowFOV = false,
    WallCheck = false,
    
    BigTorsoHitbox = false,
    HitboxScale = 3.0,
    TorsoTransparency = 0.5,
    
    BoxESP = false,
    BoxColor = Color3.fromRGB(0, 170, 255),
    
    SkeletonESP = false,
    SkeletonColor = Color3.fromRGB(0, 170, 255),
    
    PlayerHighlight = false,
    HighlightColor = Color3.fromRGB(255, 0, 0),
    
    ShowUsername = false,
    UsernameColor = Color3.fromRGB(255, 255, 255),
    
    HealthBar = false,
    InventoryEquipped = false,
    
    SpeedBoost = false,
    WalkSpeedValue = 28,
    InvisiblePlayer = false,
    NoClipTool = false,
    AutoFarmMalvavisco = false
}

local Whitelist = {}
local OriginalSizes = {}
local OriginalMassless = {}
local OriginalCanCollide = {}
local OriginalTransparency = {}
local ESPObjects = {}
local CharacterTransparencies = {}
local isAiming = false
local isSqueezeEquipped = false
local currentTouchedWall = nil

local hasDrawing = pcall(function() return Drawing.new("Circle") end)

-- ==========================================================
-- HERRAMIENTA SQUEEZE & ATRAVESAR PARED
-- ==========================================================
local function GiveSqueezeTool()
    if not Settings.NoClipTool then return end
    local backpack = LocalPlayer:FindFirstChildOfClass("Backpack")
    local char = LocalPlayer.Character
    
    if (backpack and backpack:FindFirstChild("Squeeze")) or (char and char:FindFirstChild("Squeeze")) then
        return
    end
    
    local tool = Instance.new("Tool")
    tool.Name = "Squeeze"
    tool.RequiresHandle = false
    tool.CanBeDropped = false
    
    tool.Equipped:Connect(function()
        isSqueezeEquipped = true
    end)
    
    tool.Unequipped:Connect(function()
        isSqueezeEquipped = false
        if currentTouchedWall then
            currentTouchedWall.CanCollide = true
            currentTouchedWall = nil
        end
    end)
    
    if backpack then
        tool.Parent = backpack
    end
end

local function RemoveSqueezeTool()
    isSqueezeEquipped = false
    if currentTouchedWall then
        currentTouchedWall.CanCollide = true
        currentTouchedWall = nil
    end
    local backpack = LocalPlayer:FindFirstChildOfClass("Backpack")
    local char = LocalPlayer.Character
    
    if backpack and backpack:FindFirstChild("Squeeze") then
        backpack.Squeeze:Destroy()
    end
    if char and char:FindFirstChild("Squeeze") then
        char.Squeeze:Destroy()
    end
end

LocalPlayer.CharacterAdded:Connect(function(char)
    isSqueezeEquipped = false
    currentTouchedWall = nil
    
    task.wait(0.5)
    local humanoid = char:FindFirstChildOfClass("Humanoid")
    if humanoid and Settings.SpeedBoost then
        humanoid.WalkSpeed = Settings.WalkSpeedValue
    end
    
    if Settings.NoClipTool then
        GiveSqueezeTool()
    end
end)

RunService.Stepped:Connect(function()
    if Settings.NoClipTool and isSqueezeEquipped then
        local char = LocalPlayer.Character
        if not char then return end
        
        local root = char:FindFirstChild("HumanoidRootPart")
        local humanoid = char:FindFirstChildOfClass("Humanoid")
        
        if root and humanoid then
            local moveDir = humanoid.MoveDirection
            if moveDir.Magnitude > 0 then
                local rayParams = RaycastParams.new()
                rayParams.FilterType = Enum.RaycastFilterType.Exclude
                rayParams.FilterDescendantsInstances = {char}
                
                local rayResult = workspace:Raycast(root.Position, moveDir * 2, rayParams)
                
                if rayResult and rayResult.Instance and rayResult.Instance:IsA("BasePart") then
                    local hitPart = rayResult.Instance
                    
                    if hitPart.Size.Y > 2 and math.abs(rayResult.Normal.Y) < 0.5 then
                        if currentTouchedWall and currentTouchedWall ~= hitPart then
                            currentTouchedWall.CanCollide = true
                        end
                        
                        currentTouchedWall = hitPart
                        currentTouchedWall.CanCollide = false
                        
                        root.CFrame = root.CFrame + (moveDir * 0.35)
                    end
                end
            else
                if currentTouchedWall then
                    currentTouchedWall.CanCollide = true
                    currentTouchedWall = nil
                end
            end
        end
    else
        if currentTouchedWall then
            currentTouchedWall.CanCollide = true
            currentTouchedWall = nil
        end
    end
end)

-- ==========================================================
-- INVISIBILIDAD
-- ==========================================================
local function ApplyInvisibilityToInstance(v, invisible)
    if v:IsA("BasePart") or v:IsA("Decal") or v:IsA("Texture") then
        if invisible then
            if CharacterTransparencies[v] == nil then
                CharacterTransparencies[v] = v.Transparency
            end
            v.Transparency = 1
        else
            if CharacterTransparencies[v] ~= nil then
                v.Transparency = CharacterTransparencies[v]
            else
                v.Transparency = 0
            end
        end
    end
end

local function SetPlayerInvisibility(invisible)
    local char = LocalPlayer.Character
    if not char then return end

    for _, v in pairs(char:GetDescendants()) do
        ApplyInvisibilityToInstance(v, invisible)
    end
end

LocalPlayer.CharacterAdded:Connect(function(char)
    CharacterTransparencies = {}
    task.wait(0.5)
    if Settings.InvisiblePlayer then SetPlayerInvisibility(true) end
end)

-- ==========================================================
-- HITBOX EXPANDER
-- ==========================================================
local function GetTorso(character)
    if not character then return nil end
    return character:FindFirstChild("UpperTorso") or character:FindFirstChild("Torso")
end

local function RestoreAllHitboxes()
    for _, player in pairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character then
            local torso = GetTorso(player.Character)
            if torso and OriginalSizes[torso] then
                torso.Size = OriginalSizes[torso]
                torso.Transparency = OriginalTransparency[torso] or 0
                torso.Massless = OriginalMassless[torso] or false
                torso.CanCollide = OriginalCanCollide[torso] ~= nil and OriginalCanCollide[torso] or true
            end
        end
    end
end

RunService.Heartbeat:Connect(function()
    if Settings.BigTorsoHitbox then
        for _, player in pairs(Players:GetPlayers()) do
            if player ~= LocalPlayer and player.Character and not Whitelist[player.UserId] then
                local torso = GetTorso(player.Character)
                if torso then
                    if not OriginalSizes[torso] then
                        OriginalSizes[torso] = torso.Size
                        OriginalMassless[torso] = torso.Massless
                        OriginalCanCollide[torso] = torso.CanCollide
                        OriginalTransparency[torso] = torso.Transparency
                    end
                    
                    torso.Size = OriginalSizes[torso] * Settings.HitboxScale
                    torso.Transparency = Settings.TorsoTransparency
                    torso.Massless = true
                    torso.CanCollide = false
                end
            end
        end
    else
        RestoreAllHitboxes()
    end
end)

-- ==========================================================
-- AIMBOT
-- ==========================================================
local function GetClosestTarget()
    local closestPlayer = nil
    local shortestDistance = Settings.FOV
    local mousePos = UserInputService:GetMouseLocation()

    for _, player in pairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and not Whitelist[player.UserId] then
            local char = player.Character
            if not char then continue end

            local targetPart = GetTorso(char)
            local humanoid = char:FindFirstChildOfClass("Humanoid")
            
            if targetPart and humanoid and humanoid.Health > 0 then
                local screenPos, onScreen = Camera:WorldToViewportPoint(targetPart.Position)
                
                if onScreen then
                    local distance = (Vector2.new(screenPos.X, screenPos.Y) - mousePos).Magnitude
                    if distance < shortestDistance then
                        if Settings.WallCheck then
                            local filterList = {}
                            if LocalPlayer.Character then table.insert(filterList, LocalPlayer.Character) end
                            table.insert(filterList, char)
                            
                            local rayParams = RaycastParams.new()
                            rayParams.FilterType = Enum.RaycastFilterType.Exclude
                            rayParams.FilterDescendantsInstances = filterList
                            
                            local ray = workspace:Raycast(Camera.CFrame.Position, (targetPart.Position - Camera.CFrame.Position), rayParams)
                            if ray then continue end 
                        end
                        shortestDistance = distance
                        closestPlayer = player
                    end
                end
            end
        end
    end
    return closestPlayer
end

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if not gameProcessed and input.KeyCode == Settings.Keybind then
        if Settings.Activation == "Hold" then
            isAiming = true
        elseif Settings.Activation == "Toggle" then
            isAiming = not isAiming
        end
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if input.KeyCode == Settings.Keybind and Settings.Activation == "Hold" then
        isAiming = false
    end
end)

-- ==========================================================
-- SISTEMA ESP & HIGHLIGHT
-- ==========================================================
local function CreateESP(player)
    if player == LocalPlayer or ESPObjects[player] then return end
    
    local objects = {
        Box = hasDrawing and Drawing.new("Square") or nil,
        Name = hasDrawing and Drawing.new("Text") or nil,
        Inventory = hasDrawing and Drawing.new("Text") or nil,
        HealthBar = hasDrawing and Drawing.new("Line") or nil,
        HealthBarBackground = hasDrawing and Drawing.new("Line") or nil,
        Highlight = Instance.new("Highlight"),
        Bones = {}
    }

    objects.Highlight.Name = "CipherHighlight"
    objects.Highlight.FillColor = Settings.HighlightColor
    objects.Highlight.OutlineColor = Color3.fromRGB(255, 255, 255)
    objects.Highlight.FillTransparency = 0.3
    objects.Highlight.OutlineTransparency = 0
    objects.Highlight.Enabled = false

    if objects.Box then
        objects.Box.Thickness = 1
        objects.Box.Color = Settings.BoxColor
        objects.Box.Filled = false
        objects.Box.Visible = false
    end

    if objects.Name then
        objects.Name.Size = 14
        objects.Name.Center = true
        objects.Name.Outline = true
        objects.Name.OutlineColor = Color3.fromRGB(0, 0, 0)
        objects.Name.Color = Settings.UsernameColor
        objects.Name.Visible = false
    end

    if objects.Inventory then
        objects.Inventory.Size = 13
        objects.Inventory.Center = true
        objects.Inventory.Outline = true
        objects.Inventory.OutlineColor = Color3.fromRGB(0, 0, 0)
        objects.Inventory.Color = Color3.fromRGB(255, 255, 255)
        objects.Inventory.Visible = false
    end

    if objects.HealthBarBackground then
        objects.HealthBarBackground.Thickness = 3
        objects.HealthBarBackground.Color = Color3.fromRGB(0, 0, 0)
        objects.HealthBarBackground.Visible = false
    end

    if objects.HealthBar then
        objects.HealthBar.Thickness = 2
        objects.HealthBar.Color = Color3.fromRGB(0, 255, 0)
        objects.HealthBar.Visible = false
    end

    local boneConnections = {
        {"Head", "UpperTorso"}, {"UpperTorso", "LowerTorso"},
        {"UpperTorso", "LeftUpperArm"}, {"LeftUpperArm", "LeftHand"},
        {"UpperTorso", "RightUpperArm"}, {"RightUpperArm", "RightHand"},
        {"LowerTorso", "LeftUpperLeg"}, {"LeftUpperLeg", "LeftFoot"},
        {"LowerTorso", "RightUpperLeg"}, {"RightUpperLeg", "RightFoot"},
        {"Head", "Torso"}, {"Torso", "Left Arm"}, {"Torso", "Right Arm"}, {"Torso", "Left Leg"}, {"Torso", "Right Leg"}
    }

    if hasDrawing then
        for i = 1, #boneConnections do
            local line = Drawing.new("Line")
            line.Thickness = 1.5
            line.Color = Settings.SkeletonColor
            line.Visible = false
            table.insert(objects.Bones, {Line = line, Connection = boneConnections[i]})
        end
    end

    ESPObjects[player] = objects
end

local function RemoveESP(player)
    if ESPObjects[player] then
        local obj = ESPObjects[player]
        if obj.Box then obj.Box:Remove() end
        if obj.Name then obj.Name:Remove() end
        if obj.Inventory then obj.Inventory:Remove() end
        if obj.HealthBar then obj.HealthBar:Remove() end
        if obj.HealthBarBackground then obj.HealthBarBackground:Remove() end
        if obj.Highlight then obj.Highlight:Destroy() end
        for _, boneData in pairs(obj.Bones) do boneData.Line:Remove() end
        ESPObjects[player] = nil
    end
end

for _, p in pairs(Players:GetPlayers()) do CreateESP(p) end
Players.PlayerAdded:Connect(CreateESP)
Players.PlayerRemoving:Connect(RemoveESP)

-- Visual FOV
local FOVCircle = hasDrawing and Drawing.new("Circle") or nil
if FOVCircle then
    FOVCircle.Thickness = 1.5
    FOVCircle.NumSides = 64
    FOVCircle.Filled = false
    FOVCircle.Visible = false
    FOVCircle.Color = Color3.fromRGB(0, 170, 255)
end

-- Render Bucle (Aimbot, ESP & FOV Arcoíris)
local rainbowHue = 0
RunService.RenderStepped:Connect(function()
    local mousePos = UserInputService:GetMouseLocation()

    if FOVCircle then
        if Settings.ShowFOVCircle then
            FOVCircle.Position = mousePos
            FOVCircle.Radius = Settings.FOV
            FOVCircle.Visible = true
            
            if Settings.RainbowFOV then
                rainbowHue = (rainbowHue + 0.005) % 1
                FOVCircle.Color = Color3.fromHSV(rainbowHue, 1, 1)
            else
                FOVCircle.Color = Color3.fromRGB(0, 170, 255)
            end
        else
            FOVCircle.Visible = false
        end
    end

    if Settings.AimbotEnabled and isAiming then
        local target = GetClosestTarget()
        if target and target.Character then
            local targetPart = GetTorso(target.Character)
            if targetPart then
                if Settings.AimMode == "Camera" then
                    Camera.CFrame = Camera.CFrame:Lerp(CFrame.new(Camera.CFrame.Position, targetPart.Position), 1 / Settings.Smoothness)
                elseif Settings.AimMode == "Mouse" and mousemoveabs then
                    local targetPos = Camera:WorldToViewportPoint(targetPart.Position)
                    mousemoveabs(targetPos.X, targetPos.Y)
                end
            end
        end
    end

    for player, esp in pairs(ESPObjects) do
        local char = player.Character
        if char and player ~= LocalPlayer then
            local root = char:FindFirstChild("HumanoidRootPart")
            local head = char:FindFirstChild("Head")
            local humanoid = char:FindFirstChildOfClass("Humanoid")

            if Settings.PlayerHighlight and esp.Highlight then
                esp.Highlight.Parent = char
                esp.Highlight.FillColor = Settings.HighlightColor
                esp.Highlight.Enabled = true
            elseif esp.Highlight then
                esp.Highlight.Enabled = false
            end

            if root and head then
                local rootPos, onScreen = Camera:WorldToViewportPoint(root.Position)

                if onScreen then
                    local headPos = Camera:WorldToViewportPoint(head.Position + Vector3.new(0, 0.5, 0))
                    local height = math.abs(headPos.Y - rootPos.Y) * 2
                    local width = height / 1.8

                    if Settings.BoxESP and esp.Box then
                        esp.Box.Size = Vector2.new(width, height)
                        esp.Box.Position = Vector2.new(rootPos.X - width / 2, rootPos.Y - height / 2)
                        esp.Box.Color = Settings.BoxColor
                        esp.Box.Visible = true
                    elseif esp.Box then
                        esp.Box.Visible = false
                    end

                    if Settings.ShowUsername and esp.Name then
                        local namePos = Camera:WorldToViewportPoint(head.Position + Vector3.new(0, 1.2, 0))
                        esp.Name.Text = player.Name
                        esp.Name.Position = Vector2.new(namePos.X, namePos.Y - 15)
                        esp.Name.Color = Settings.UsernameColor
                        esp.Name.Visible = true
                    elseif esp.Name then
                        esp.Name.Visible = false
                    end

                    if Settings.InventoryEquipped and esp.Inventory then
                        local items = {}
                        for _, item in pairs(char:GetChildren()) do
                            if item:IsA("Tool") then table.insert(items, item.Name) end
                        end
                        if player:FindFirstChild("Backpack") then
                            for _, item in pairs(player.Backpack:GetChildren()) do
                                if item:IsA("Tool") then table.insert(items, item.Name) end
                            end
                        end

                        if #items > 0 then
                            esp.Inventory.Text = table.concat(items, "\n")
                            local invPos = Camera:WorldToViewportPoint(head.Position + Vector3.new(0, 1.8, 0))
                            local offsetY = Settings.ShowUsername and 35 or 20
                            esp.Inventory.Position = Vector2.new(invPos.X, invPos.Y - offsetY)
                            esp.Inventory.Visible = true
                        else
                            esp.Inventory.Visible = false
                        end
                    elseif esp.Inventory then
                        esp.Inventory.Visible = false
                    end

                    if Settings.HealthBar and esp.HealthBar and humanoid then
                        local barX = (rootPos.X - width / 2) - 6
                        local topY = rootPos.Y - height / 2
                        local bottomY = rootPos.Y + height / 2
                        
                        local healthPercent = math.clamp(humanoid.Health / humanoid.MaxHealth, 0, 1)
                        local healthHeight = height * healthPercent

                        esp.HealthBarBackground.From = Vector2.new(barX, topY)
                        esp.HealthBarBackground.To = Vector2.new(barX, bottomY)
                        esp.HealthBarBackground.Visible = true

                        esp.HealthBar.From = Vector2.new(barX, bottomY)
                        esp.HealthBar.To = Vector2.new(barX, bottomY - healthHeight)
                        esp.HealthBar.Color = Color3.fromRGB(255 - (255 * healthPercent), 255 * healthPercent, 0)
                        esp.HealthBar.Visible = true
                    else
                        if esp.HealthBar then esp.HealthBar.Visible = false end
                        if esp.HealthBarBackground then esp.HealthBarBackground.Visible = false end
                    end

                    if Settings.SkeletonESP then
                        for _, boneData in pairs(esp.Bones) do
                            local partA = char:FindFirstChild(boneData.Connection[1])
                            local partB = char:FindFirstChild(boneData.Connection[2])

                            if partA and partB then
                                local posA, visA = Camera:WorldToViewportPoint(partA.Position)
                                local posB, visB = Camera:WorldToViewportPoint(partB.Position)

                                if visA and visB then
                                    boneData.Line.From = Vector2.new(posA.X, posA.Y)
                                    boneData.Line.To = Vector2.new(posB.X, posB.Y)
                                    boneData.Line.Color = Settings.SkeletonColor
                                    boneData.Line.Visible = true
                                else
                                    boneData.Line.Visible = false
                                end
                            else
                                boneData.Line.Visible = false
                            end
                        end
                    else
                        for _, boneData in pairs(esp.Bones) do boneData.Line.Visible = false end
                    end
                else
                    if esp.Box then esp.Box.Visible = false end
                    if esp.Name then esp.Name.Visible = false end
                    if esp.Inventory then esp.Inventory.Visible = false end
                    if esp.HealthBar then esp.HealthBar.Visible = false end
                    if esp.HealthBarBackground then esp.HealthBarBackground.Visible = false end
                    for _, boneData in pairs(esp.Bones) do boneData.Line.Visible = false end
                end
            end
        else
            if esp.Box then esp.Box.Visible = false end
            if esp.Name then esp.Name.Visible = false end
            if esp.Inventory then esp.Inventory.Visible = false end
            if esp.HealthBar then esp.HealthBar.Visible = false end
            if esp.HealthBarBackground then esp.HealthBarBackground.Visible = false end
            if esp.Highlight then esp.Highlight.Enabled = false end
            for _, boneData in pairs(esp.Bones) do boneData.Line.Visible = false end
        end
    end
end)

-- ==========================================================
-- INTERFAZ GRÁFICA (RAYFIELD UI - TEMA OCEAN)
-- ==========================================================
local Window = Rayfield:CreateWindow({
   Name = "panel squeeze",
   LoadingTitle = "Cargando Panel Squeeze...",
   LoadingSubtitle = "por SOSA",
   ConfigurationSaving = { Enabled = false },
   KeySystem = false,
   Keybind = "F1",
   Theme = "Ocean"
})

local CombatTab = Window:CreateTab("Combate", "crosshair")
local VisualsTab = Window:CreateTab("Visuales", "eye")
local PlayerTab = Window:CreateTab("Jugador", "user")
local FarmTab = Window:CreateTab("Auto Farm", "wheat")

-- --- COMBAT TAB ---
CombatTab:CreateSection("— AIMBOT —")

CombatTab:CreateToggle({
   Name = "Activar Aimbot",
   CurrentValue = Settings.AimbotEnabled,
   Callback = function(Value) Settings.AimbotEnabled = Value end,
})

CombatTab:CreateKeybind({
   Name = "Tecla de Activación",
   CurrentKeybind = "E",
   HoldToInteract = false,
   Callback = function(Keybind) 
       if type(Keybind) == "string" then
           Settings.Keybind = Enum.KeyCode[Keybind] or Enum.KeyCode.E
       elseif typeof(Keybind) == "EnumItem" then
           Settings.Keybind = Keybind
       end
   end,
})

CombatTab:CreateDropdown({
   Name = "Modo de Apuntado",
   Options = {"Camera", "Mouse"},
   CurrentOption = {"Camera"},
   MultipleOptions = false,
   Callback = function(Option) Settings.AimMode = Option[1] end,
})

CombatTab:CreateDropdown({
   Name = "Tipo de Activación",
   Options = {"Hold", "Toggle"},
   CurrentOption = {"Hold"},
   MultipleOptions = false,
   Callback = function(Option) Settings.Activation = Option[1] end,
})

CombatTab:CreateSlider({
   Name = "Radio del FOV",
   Range = {10, 500},
   Increment = 5,
   Suffix = " px",
   CurrentValue = Settings.FOV,
   Callback = function(Value) Settings.FOV = Value end,
})

CombatTab:CreateSlider({
   Name = "Suavizado (Smoothness)",
   Range = {1, 20},
   Increment = 1,
   Suffix = "",
   CurrentValue = Settings.Smoothness,
   Callback = function(Value) Settings.Smoothness = Value end,
})

CombatTab:CreateToggle({
   Name = "Mostrar Círculo FOV",
   CurrentValue = Settings.ShowFOVCircle,
   Callback = function(Value) Settings.ShowFOVCircle = Value end,
})

CombatTab:CreateToggle({
   Name = "FOV Color Arcoíris",
   CurrentValue = Settings.RainbowFOV,
   Callback = function(Value) Settings.RainbowFOV = Value end,
})

CombatTab:CreateToggle({
   Name = "Verificar Pared (Wall Check)",
   CurrentValue = Settings.WallCheck,
   Callback = function(Value) Settings.WallCheck = Value end,
})

CombatTab:CreateSection("— HITBOX (TORSO) —")

CombatTab:CreateToggle({
   Name = "Expandir Hitbox de Torso",
   CurrentValue = Settings.BigTorsoHitbox,
   Callback = function(Value) 
       Settings.BigTorsoHitbox = Value 
       if not Value then RestoreAllHitboxes() end
   end,
})

CombatTab:CreateSlider({
   Name = "Escala de Hitbox",
   Range = {1, 10},
   Increment = 0.5,
   Suffix = "x",
   CurrentValue = Settings.HitboxScale,
   Callback = function(Value) Settings.HitboxScale = Value end,
})

CombatTab:CreateSlider({
   Name = "Transparencia de Hitbox",
   Range = {0, 1},
   Increment = 0.1,
   Suffix = "",
   CurrentValue = Settings.TorsoTransparency,
   Callback = function(Value) Settings.TorsoTransparency = Value end,
})

-- --- VISUALS TAB ---
VisualsTab:CreateSection("— ESP & VISUALES —")

VisualsTab:CreateToggle({
   Name = "ESP de Caja (Box)",
   CurrentValue = Settings.BoxESP,
   Callback = function(Value) Settings.BoxESP = Value end,
})
VisualsTab:CreateColorPicker({
    Name = "Color de Caja",
    Color = Settings.BoxColor,
    Callback = function(Value) Settings.BoxColor = Value end
})

VisualsTab:CreateToggle({
   Name = "ESP de Esqueleto",
   CurrentValue = Settings.SkeletonESP,
   Callback = function(Value) Settings.SkeletonESP = Value end,
})
VisualsTab:CreateColorPicker({
    Name = "Color de Esqueleto",
    Color = Settings.SkeletonColor,
    Callback = function(Value) Settings.SkeletonColor = Value end
})

VisualsTab:CreateToggle({
   Name = "Resaltar Jugadores (Highlight)",
   CurrentValue = Settings.PlayerHighlight,
   Callback = function(Value) Settings.PlayerHighlight = Value end,
})
VisualsTab:CreateColorPicker({
    Name = "Color de Resaltado",
    Color = Settings.HighlightColor,
    Callback = function(Value) Settings.HighlightColor = Value end
})

VisualsTab:CreateToggle({
   Name = "Mostrar Nombres",
   CurrentValue = Settings.ShowUsername,
   Callback = function(Value) Settings.ShowUsername = Value end,
})

VisualsTab:CreateToggle({
   Name = "Barra de Vida",
   CurrentValue = Settings.HealthBar,
   Callback = function(Value) Settings.HealthBar = Value end,
})

VisualsTab:CreateToggle({
   Name = "Mostrar Inventario",
   CurrentValue = Settings.InventoryEquipped,
   Callback = function(Value) Settings.InventoryEquipped = Value end,
})

-- --- PLAYER TAB ---
PlayerTab:CreateSection("— MOVIMIENTO —")

PlayerTab:CreateToggle({
   Name = "Aumentar Velocidad",
   CurrentValue = Settings.SpeedBoost,
   Callback = function(Value) 
       Settings.SpeedBoost = Value 
       if LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Humanoid") then
           LocalPlayer.Character:FindFirstChildOfClass("Humanoid").WalkSpeed = Value and Settings.WalkSpeedValue or 16
       end
   end,
})

PlayerTab:CreateSlider({
   Name = "Valor de Velocidad",
   Range = {16, 120},
   Increment = 2,
   Suffix = " speed",
   CurrentValue = Settings.WalkSpeedValue,
   Callback = function(Value) 
       Settings.WalkSpeedValue = Value
       if Settings.SpeedBoost and LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Humanoid") then
           LocalPlayer.Character:FindFirstChildOfClass("Humanoid").WalkSpeed = Value
       end
   end,
})

PlayerTab:CreateToggle({
   Name = "Jugador Invisible",
   CurrentValue = Settings.InvisiblePlayer,
   Callback = function(Value) 
       Settings.InvisiblePlayer = Value
       SetPlayerInvisibility(Value)
   end,
})

PlayerTab:CreateToggle({
   Name = "Atravesar Pared (Herramienta Squeeze)",
   CurrentValue = Settings.NoClipTool,
   Callback = function(Value) 
       Settings.NoClipTool = Value
       if Value then
           GiveSqueezeTool()
       else
           RemoveSqueezeTool()
       end
   end,
})

-- --- FARM TAB ---
FarmTab:CreateSection("— MALVAVISCO —")

FarmTab:CreateToggle({
   Name = "Auto Farm Malvavisco",
   CurrentValue = Settings.AutoFarmMalvavisco,
   Callback = function(Value)
       Settings.AutoFarmMalvavisco = Value
       if Value then
           task.spawn(function()
               while Settings.AutoFarmMalvavisco do
                   task.wait(1)
               end
           end)
       end
   end,
})
