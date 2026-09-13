local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local CoreGui = game:GetService("CoreGui")
local Lighting = game:GetService("Lighting")
local HttpService = game:GetService("HttpService")

local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera

local ToggleKeybind = Enum.KeyCode.RightShift
local AimModeKeybind = Enum.KeyCode.E
local isListeningForKey = false
local isListeningForAimKey = false
local isRMBDown = false

local ContainerParent
pcall(function()
    ContainerParent = gethui()
end)
if not ContainerParent then
    local success, err = pcall(function()
        return CoreGui:FindFirstChild("RobloxGui") or CoreGui
    end)
    ContainerParent = success and err or LocalPlayer:WaitForChild("PlayerGui")
end

if ContainerParent:FindFirstChild("REDJOHN_HUB") then
    ContainerParent:FindFirstChild("REDJOHN_HUB"):Destroy()
end

local DefaultConfig = {
    AimEnabled = true,
    AimMode = "Hold RMB",
    BotLockEnabled = true,
    WallCheckEnabled = false,
    PredictionEnabled = true,
    AimSmoothness = 0.2,
    
    NoRecoilEnabled = true,
    FireRateEnabled = false,
    FireRateMultiplier = 1.5,
    ToggleKeybindName = "RightShift",
    AimModeKeybindName = "E",
    
    PlayerESPEnabled = true,
    BotESPEnabled = true,
    ItemBoxESPEnabled = true,
    ExitESPEnabled = true,
    CorpseESPEnabled = true,
    FullBrightEnabled = false,
    FullBrightIntensity = 3.0,
    ShowFOVCircle = true,
    
    ESPTextSize = 14, -- ขนาดตัวหนังสือ ESP
    UITextSize = 13,   -- ขนาดตัวหนังสือ UI เมนู

    FOV = 300,
    PlayerESPDistance = 5000,
    BotESPDistance = 5000,
    ItemBoxESPDistance = 5000,
    ExitESPDistance = 5000,
    CorpseESPDistance = 5000
}

local Config = {}
for k, v in pairs(DefaultConfig) do Config[k] = v end

local ConfigFileName = "REDJOHN_HUB_Config.json"
local UI_Elements = { Toggles = {}, Sliders = {}, Buttons = {} }
local DynamicUITexts = {}
local KeybindButtonRef = nil
local AimKeybindButtonRef = nil

local Connections = {}

local OriginalLighting = {
    Brightness = Lighting.Brightness,
    ClockTime = Lighting.ClockTime,
    FogEnd = Lighting.FogEnd,
    GlobalShadows = Lighting.GlobalShadows,
    Ambient = Lighting.Ambient,
    OutdoorAmbient = Lighting.OutdoorAmbient
}

local TargetCache = {
    Bots = {},
    ItemBoxes = {},
    Exits = {},
    Corpses = {}
}

local ESPCache = {}

local function SaveConfig()
    pcall(function()
        if writefile then
            Config.ToggleKeybindName = ToggleKeybind.Name
            Config.AimModeKeybindName = AimModeKeybind.Name
            local data = HttpService:JSONEncode(Config)
            writefile(ConfigFileName, data)
        end
    end)
end

local function LoadConfig()
    pcall(function()
        if readfile and isfile and isfile(ConfigFileName) then
            local success, result = pcall(function()
                return HttpService:JSONDecode(readfile(ConfigFileName))
            end)
            if success and type(result) == "table" then
                for k, v in pairs(result) do
                    if Config[k] ~= nil then Config[k] = v end
                end
                if Config.ToggleKeybindName then
                    local successKey, keyCode = pcall(function()
                        return Enum.KeyCode[Config.ToggleKeybindName]
                    end)
                    if successKey and keyCode then
                        ToggleKeybind = keyCode
                        if KeybindButtonRef then
                            KeybindButtonRef.Text = "UI Key: " .. tostring(ToggleKeybind.Name)
                        end
                    end
                end
                if Config.AimModeKeybindName then
                    local successKey, keyCode = pcall(function()
                        return Enum.KeyCode[Config.AimModeKeybindName]
                    end)
                    if successKey and keyCode then
                        AimModeKeybind = keyCode
                        if AimKeybindButtonRef then
                            AimKeybindButtonRef.Text = "Mode Key: " .. tostring(AimModeKeybind.Name)
                        end
                    end
                end
            end
        end
    end)
end

local function DeleteConfig()
    pcall(function()
        if delfile and isfile and isfile(ConfigFileName) then
            delfile(ConfigFileName)
        end
    end)
    for k, v in pairs(DefaultConfig) do 
        Config[k] = v 
    end
    ToggleKeybind = Enum.KeyCode.RightShift
    AimModeKeybind = Enum.KeyCode.E
    if KeybindButtonRef then
        KeybindButtonRef.Text = "UI Key: " .. tostring(ToggleKeybind.Name)
    end
    if AimKeybindButtonRef then
        AimKeybindButtonRef.Text = "Mode Key: " .. tostring(AimModeKeybind.Name)
    end
end

LoadConfig()

local function UpdateAllUITextSizes(size)
    for _, obj in ipairs(DynamicUITexts) do
        if obj and obj.Parent then
            obj.TextSize = size
        end
    end
end

local function ApplyFullBright()
    pcall(function()
        if Config.FullBrightEnabled then
            Lighting.Brightness = Config.FullBrightIntensity
            Lighting.ClockTime = 14
            Lighting.FogEnd = 1000000
            Lighting.GlobalShadows = false
            Lighting.Ambient = Color3.fromRGB(255, 255, 255)
            Lighting.OutdoorAmbient = Color3.fromRGB(255, 255, 255)
        else
            Lighting.Brightness = OriginalLighting.Brightness
            Lighting.ClockTime = OriginalLighting.ClockTime
            Lighting.FogEnd = OriginalLighting.FogEnd
            Lighting.GlobalShadows = OriginalLighting.GlobalShadows
            Lighting.Ambient = OriginalLighting.Ambient
            Lighting.OutdoorAmbient = OriginalLighting.OutdoorAmbient
        end
    end)
end

table.insert(Connections, Lighting.Changed:Connect(function()
    if Config.FullBrightEnabled then
        ApplyFullBright()
    end
end))

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "REDJOHN_HUB"
ScreenGui.ResetOnSpawn = false
ScreenGui.Parent = ContainerParent

local FOVCircle = Instance.new("Frame", ScreenGui)
FOVCircle.Name = "FOVCircle"
FOVCircle.BackgroundTransparency = 1
FOVCircle.AnchorPoint = Vector2.new(0.5, 0.5)
FOVCircle.Visible = Config.ShowFOVCircle

local FOVStroke = Instance.new("UIStroke", FOVCircle)
FOVStroke.Color = Color3.fromRGB(255, 255, 255)
FOVStroke.Thickness = 1.5
Instance.new("UICorner", FOVCircle).CornerRadius = UDim.new(1, 0)

local function GetMainPart(model)
    if not model then return nil end
    if model:IsA("BasePart") then return model end
    return model:FindFirstChild("Head") 
        or model:FindFirstChild("HumanoidRootPart") 
        or model.PrimaryPart 
        or model:FindFirstChildOfClass("BasePart")
end

local function GetItemName(obj)
    if not obj then return nil end
    if obj:FindFirstChild("ItemName") and obj.ItemName:IsA("StringValue") then
        return obj.ItemName.Value
    elseif obj:GetAttribute("ItemName") then
        return tostring(obj:GetAttribute("ItemName"))
    elseif obj:GetAttribute("Name") then
        return tostring(obj:GetAttribute("Name"))
    end
    return obj.Name
end

local function GetPlayerHeldItem(player)
    if not player then return "Nothing" end
    local itemsFound = {}
    local addedNames = {}

    local function addItem(name)
        if name and name ~= "" and not addedNames[name] then
            addedNames[name] = true
            table.insert(itemsFound, name)
        end
    end

    local backpack = player:FindFirstChildOfClass("Backpack") or player:FindFirstChild("Backpack") or player:FindFirstChild("Inventory")
    if backpack then
        for _, item in ipairs(backpack:GetChildren()) do
            local itemName = GetItemName(item) or item.Name
            addItem(itemName)
        end
    end

    local char = player.Character
    if char then
        for _, child in ipairs(char:GetChildren()) do
            if child:IsA("Tool") then
                addItem(GetItemName(child) or child.Name)
            elseif child:IsA("Model") and not child:FindFirstChildOfClass("Humanoid") then
                if not (child:IsA("Accessory") or child:IsA("Hat") or child:IsA("Clothing") or child:IsA("ShirtGraphic")) then
                    local childName = child.Name:lower()
                    local isClothing = childName:find("shirt") or childName:find("pants") or childName:find("vest") 
                                    or childName:find("armor") or childName:find("helmet") or childName:find("cloth") 
                                    or childName:find("bag") or childName:find("backpack") or childName:find("suit")
                    if not isClothing then
                        addItem(GetItemName(child) or child.Name)
                    end
                end
            end
        end

        local inventoryFolder = char:FindFirstChild("Inventory") or char:FindFirstChild("Weapons") or char:FindFirstChild("Slots")
        if inventoryFolder then
            for _, val in ipairs(inventoryFolder:GetChildren()) do
                if val:IsA("StringValue") and val.Value ~= "" then
                    addItem(val.Value)
                elseif val:IsA("ObjectValue") and val.Value then
                    addItem(val.Value.Name)
                end
            end
        end
    end

    if #itemsFound > 0 then
        return table.concat(itemsFound, ", ")
    end

    return "Nothing"
end

local function Apply3DESP(model, color)
    if not model then return nil end
    
    local data = ESPCache[model]
    if not data then
        local highlight = Instance.new("Highlight")
        highlight.Name = "REDJOHN_3DHighlight"
        highlight.Adornee = model
        highlight.FillTransparency = 0.6
        highlight.OutlineTransparency = 0.1
        highlight.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
        highlight.Parent = model

        local mainPart = GetMainPart(model)
        local tagText = nil
        local bb = nil

        if mainPart then
            bb = Instance.new("BillboardGui")
            bb.Name = "REDJOHN_3DTag"
            bb.Size = UDim2.fromOffset(300, 70)
            bb.StudsOffset = Vector3.new(0, 3, 0)
            bb.AlwaysOnTop = true

            tagText = Instance.new("TextLabel", bb)
            tagText.Name = "TagText"
            tagText.Size = UDim2.new(1, 0, 1, 0)
            tagText.BackgroundTransparency = 1
            tagText.TextStrokeTransparency = 0.3
            tagText.TextSize = Config.ESPTextSize
            tagText.Font = Enum.Font.GothamBold
            bb.Parent = mainPart
        end

        data = { Highlight = highlight, Billboard = bb, TagText = tagText }
        ESPCache[model] = data
        
        table.insert(Connections, model.Destroying:Connect(function()
            ESPCache[model] = nil
        end))
    end

    data.Highlight.Enabled = true
    data.Highlight.FillColor = color
    data.Highlight.OutlineColor = color

    if data.Billboard then
        data.Billboard.Enabled = true
        data.TagText.TextColor3 = color
        data.TagText.TextSize = Config.ESPTextSize
    end

    return data.TagText
end

local function Disable3DESP(model)
    local data = ESPCache[model]
    if data then
        if data.Highlight then data.Highlight.Enabled = false end
        if data.Billboard then data.Billboard.Enabled = false end
    end
end

local function DestroyAllESP()
    for model, data in pairs(ESPCache) do
        if data.Highlight then data.Highlight:Destroy() end
        if data.Billboard then data.Billboard:Destroy() end
    end
    table.clear(ESPCache)
end

local function IsItemOnPlayer(obj)
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr.Character and obj:IsDescendantOf(plr.Character) then
            return true
        end
    end
    return false
end

local function ClassifyAndAddObject(obj)
    if not obj or IsItemOnPlayer(obj) then return end
    local name = obj.Name:lower()
    local isDoor = name:find("door") or name:find("gate") or name:find("entrance")
    local isProp = name:find("water") or name:find("bottle") or name:find("box") or name:find("crate") or name:find("container") or name:find("loot")

    if (obj:IsA("Model") or obj:IsA("BasePart")) then
        local hum = obj:FindFirstChildOfClass("Humanoid")
        local plr = Players:GetPlayerFromCharacter(obj)

        if hum and not plr and not isProp then
            if hum.Health > 0 then
                table.insert(TargetCache.Bots, obj)
            else
                table.insert(TargetCache.Corpses, obj)
            end
        elseif not hum and not plr then
            if isProp or obj:IsA("Tool") or name:find("item") or name:find("drop") or name:find("pickup") or name:find("ammo") or name:find("weapon") then
                table.insert(TargetCache.ItemBoxes, obj)
            elseif not isDoor and (name:find("safezone") or name:find("evac") or name:find("extract") or name:find("spawnzone")) then
                table.insert(TargetCache.Exits, obj)
            end
        end
    end
end

local function InitialScan()
    table.clear(TargetCache.Bots)
    table.clear(TargetCache.ItemBoxes)
    table.clear(TargetCache.Exits)
    table.clear(TargetCache.Corpses)

    for _, obj in ipairs(workspace:GetDescendants()) do
        ClassifyAndAddObject(obj)
    end
end

InitialScan()

table.insert(Connections, workspace.DescendantAdded:Connect(function(child)
    ClassifyAndAddObject(child)
end))

table.insert(Connections, UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if input.UserInputType == Enum.UserInputType.MouseButton2 then
        isRMBDown = true
    end

    if isListeningForKey then
        if input.UserInputType == Enum.UserInputType.Keyboard then
            ToggleKeybind = input.KeyCode
            isListeningForKey = false
            if KeybindButtonRef then
                KeybindButtonRef.Text = "UI Key: " .. tostring(ToggleKeybind.Name)
            end
            SaveConfig()
        end
        return
    end

    if isListeningForAimKey then
        if input.UserInputType == Enum.UserInputType.Keyboard then
            AimModeKeybind = input.KeyCode
            isListeningForAimKey = false
            if AimKeybindButtonRef then
                AimKeybindButtonRef.Text = "Mode Key: " .. tostring(AimModeKeybind.Name)
            end
            SaveConfig()
        end
        return
    end

    if input.KeyCode == ToggleKeybind then
        ScreenGui.Enabled = not ScreenGui.Enabled
    elseif input.KeyCode == AimModeKeybind then
        if Config.AimMode == "Hold RMB" then
            Config.AimMode = "Auto Lock (Always)"
        else
            Config.AimMode = "Hold RMB"
        end
        if UI_Elements.Buttons["AimModeBtn"] then
            UI_Elements.Buttons["AimModeBtn"].Text = "Aim Mode: " .. Config.AimMode
        end
        SaveConfig()
    end
end))

table.insert(Connections, UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton2 then
        isRMBDown = false
    end
end))

local function IsVisible(targetPart)
    if not Config.WallCheckEnabled then return true end
    local origin = Camera.CFrame.Position
    local direction = (targetPart.Position - origin)
    local raycastParams = RaycastParams.new()
    raycastParams.FilterType = Enum.RaycastFilterType.Exclude
    raycastParams.FilterDescendantsInstances = { Camera, LocalPlayer.Character }
    
    local result = workspace:Raycast(origin, direction, raycastParams)
    return result == nil or result.Instance:IsDescendantOf(targetPart.Parent)
end

local function GetClosestTarget()
    local closestTarget = nil
    local shortestDistance = Config.FOV
    local mousePos = UserInputService:GetMouseLocation()

    if Config.BotLockEnabled then
        for i = #TargetCache.Bots, 1, -1 do
            local bot = TargetCache.Bots[i]
            if bot and bot.Parent then
                local hum = bot:FindFirstChildOfClass("Humanoid")
                if not hum or hum.Health > 0 then
                    local head = bot:FindFirstChild("Head") or GetMainPart(bot)
                    if head then
                        local screenPos, onScreen = Camera:WorldToViewportPoint(head.Position)
                        if onScreen then
                            local distFromMouse = (Vector2.new(screenPos.X, screenPos.Y) - mousePos).Magnitude
                            if distFromMouse < shortestDistance and IsVisible(head) then
                                shortestDistance = distFromMouse
                                closestTarget = head
                            end
                        end
                    end
                end
            else
                table.remove(TargetCache.Bots, i)
            end
        end
    end

    if Config.AimEnabled and not closestTarget then
        for _, player in ipairs(Players:GetPlayers()) do
            if player ~= LocalPlayer and player.Character then
                local char = player.Character
                local hum = char:FindFirstChildOfClass("Humanoid")
                local targetPart = char:FindFirstChild("Head") or char:FindFirstChild("HumanoidRootPart")

                if hum and hum.Health > 0 and targetPart then
                    local screenPos, onScreen = Camera:WorldToViewportPoint(targetPart.Position)
                    if onScreen then
                        local distFromMouse = (Vector2.new(screenPos.X, screenPos.Y) - mousePos).Magnitude
                        if distFromMouse < shortestDistance and IsVisible(targetPart) then
                            shortestDistance = distFromMouse
                            closestTarget = targetPart
                        end
                    end
                end
            end
        end
    end

    return closestTarget
end

local BG_Main = Color3.fromRGB(12, 12, 16)
local Card_BG = Color3.fromRGB(18, 18, 24)
local Stroke_Color = Color3.fromRGB(40, 40, 55)
local Text_Main = Color3.fromRGB(245, 245, 250)
local Text_Sub = Color3.fromRGB(150, 150, 170)

local Main = Instance.new("Frame", ScreenGui)
Main.Size = UDim2.fromOffset(940, 720)
Main.AnchorPoint = Vector2.new(0.5, 0.5)
Main.Position = UDim2.new(0.5, 0, 0.5, 0)
Main.BackgroundColor3 = BG_Main
Main.BorderSizePixel = 0
Main.Active = true
Main.ClipsDescendants = true
Instance.new("UICorner", Main).CornerRadius = UDim.new(0, 16)
local MainStroke = Instance.new("UIStroke", Main)
MainStroke.Thickness = 1.5
MainStroke.Color = Stroke_Color

local dragging, dragStart, startPos
Main.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        dragging = true
        dragStart = input.Position
        startPos = Main.Position
    end
end)
UserInputService.InputChanged:Connect(function(input)
    if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
        local delta = input.Position - dragStart
        Main.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
    end
end)
UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        dragging = false
    end
end)

local Header = Instance.new("Frame", Main)
Header.Size = UDim2.new(1, 0, 0, 75)
Header.BackgroundColor3 = Color3.fromRGB(20, 20, 28)
Header.BorderSizePixel = 0
Header.ClipsDescendants = true
Instance.new("UICorner", Header).CornerRadius = UDim.new(0, 16)

local Title = Instance.new("TextLabel", Header)
Title.Size = UDim2.new(1, -380, 1, 0)
Title.Position = UDim2.fromOffset(24, 0)
Title.BackgroundTransparency = 1
Title.Text = "REDJOHN_HUB V2"
Title.TextColor3 = Text_Main
Title.TextSize = 18
Title.Font = Enum.Font.GothamBold
Title.TextXAlignment = Enum.TextXAlignment.Left

local KeybindButton = Instance.new("TextButton", Header)
KeybindButton.Size = UDim2.fromOffset(110, 36)
KeybindButton.Position = UDim2.new(1, -310, 0.5, -18)
KeybindButton.BackgroundColor3 = Card_BG
KeybindButton.Text = "UI Key: " .. tostring(ToggleKeybind.Name)
KeybindButton.TextColor3 = Text_Main
KeybindButton.TextSize = 11
KeybindButton.Font = Enum.Font.GothamMedium
Instance.new("UICorner", KeybindButton).CornerRadius = UDim.new(0, 10)
KeybindButtonRef = KeybindButton

KeybindButton.MouseButton1Click:Connect(function()
    isListeningForKey = true
    KeybindButton.Text = "Press Key..."
end)

local AimKeybindButton = Instance.new("TextButton", Header)
AimKeybindButton.Size = UDim2.fromOffset(110, 36)
AimKeybindButton.Position = UDim2.new(1, -195, 0.5, -18)
AimKeybindButton.BackgroundColor3 = Card_BG
AimKeybindButton.Text = "Mode Key: " .. tostring(AimModeKeybind.Name)
AimKeybindButton.TextColor3 = Text_Main
AimKeybindButton.TextSize = 11
AimKeybindButton.Font = Enum.Font.GothamMedium
Instance.new("UICorner", AimKeybindButton).CornerRadius = UDim.new(0, 10)
AimKeybindButtonRef = AimKeybindButton

AimKeybindButton.MouseButton1Click:Connect(function()
    isListeningForAimKey = true
    AimKeybindButton.Text = "Press Key..."
end)

local Minimize = Instance.new("TextButton", Header)
Minimize.Size = UDim2.fromOffset(36, 36)
Minimize.Position = UDim2.new(1, -80, 0.5, -18)
Minimize.BackgroundColor3 = Card_BG
Minimize.Text = "-"
Minimize.TextColor3 = Text_Main
Minimize.TextSize = 16
Minimize.Font = Enum.Font.GothamBold
Instance.new("UICorner", Minimize).CornerRadius = UDim.new(0, 10)

local Close = Instance.new("TextButton", Header)
Close.Size = UDim2.fromOffset(36, 36)
Close.Position = UDim2.new(1, -38, 0.5, -18)
Close.BackgroundColor3 = Color3.fromRGB(45, 20, 25)
Close.Text = "x"
Close.TextColor3 = Color3.fromRGB(255, 100, 100)
Close.TextSize = 14
Close.Font = Enum.Font.GothamBold
Instance.new("UICorner", Close).CornerRadius = UDim.new(0, 10)

local function UnloadScript()
    Config.FullBrightEnabled = false
    ApplyFullBright()
    
    for _, conn in ipairs(Connections) do
        if conn and conn.Connected then
            conn:Disconnect()
        end
    end
    table.clear(Connections)
    
    DestroyAllESP()
    
    if ScreenGui then
        ScreenGui:Destroy()
    end
end

Close.MouseButton1Click:Connect(UnloadScript)

local ContentContainer = Instance.new("ScrollingFrame", Main)
ContentContainer.Size = UDim2.new(1, -32, 1, -145)
ContentContainer.Position = UDim2.fromOffset(16, 90)
ContentContainer.BackgroundTransparency = 1
ContentContainer.CanvasSize = UDim2.new(0, 0, 0, 0)
ContentContainer.AutomaticCanvasSize = Enum.AutomaticSize.Y
ContentContainer.ScrollBarThickness = 4
ContentContainer.ScrollBarImageColor3 = Color3.fromRGB(70, 70, 90)

local Footer = Instance.new("Frame", Main)
Footer.Size = UDim2.new(1, -32, 0, 45)
Footer.Position = UDim2.new(0, 16, 1, -55)
Footer.BackgroundTransparency = 1

local isMinimized = false
Minimize.MouseButton1Click:Connect(function()
    isMinimized = not isMinimized
    ContentContainer.Visible = not isMinimized
    Footer.Visible = not isMinimized
    Main.Size = isMinimized and UDim2.fromOffset(940, 75) or UDim2.fromOffset(940, 720)
end)

local LeftCol = Instance.new("Frame", ContentContainer)
LeftCol.Size = UDim2.new(0.488, 0, 0, 0)
LeftCol.AutomaticSize = Enum.AutomaticSize.Y
LeftCol.BackgroundTransparency = 1
local leftLayout = Instance.new("UIListLayout", LeftCol)
leftLayout.Padding = UDim.new(0, 10)

local RightCol = Instance.new("Frame", ContentContainer)
RightCol.Size = UDim2.new(0.488, 0, 0, 0)
RightCol.Position = UDim2.new(0.512, 0, 0, 0)
RightCol.AutomaticSize = Enum.AutomaticSize.Y
RightCol.BackgroundTransparency = 1
local rightLayout = Instance.new("UIListLayout", RightCol)
rightLayout.Padding = UDim.new(0, 10)

local function CreateToggle(text, idKey, parent, callback)
    local btn = Instance.new("TextButton", parent)
    btn.Size = UDim2.new(1, 0, 0, 44)
    btn.BackgroundColor3 = Card_BG
    btn.Text = ""
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 10)
    local stroke = Instance.new("UIStroke", btn)
    stroke.Thickness = 1
    stroke.Color = Stroke_Color
    
    local label = Instance.new("TextLabel", btn)
    label.Size = UDim2.new(1, -70, 1, 0)
    label.Position = UDim2.fromOffset(16, 0)
    label.BackgroundTransparency = 1
    label.Text = text
    label.TextColor3 = Text_Main    
    label.Font = Enum.Font.GothamMedium
    label.TextSize = Config.UITextSize
    label.TextXAlignment = Enum.TextXAlignment.Left
    table.insert(DynamicUITexts, label)

    local switchBg = Instance.new("Frame", btn)
    switchBg.Size = UDim2.fromOffset(42, 22)
    switchBg.Position = UDim2.new(1, -54, 0.5, -11)
    Instance.new("UICorner", switchBg).CornerRadius = UDim.new(1, 0)

    local circle = Instance.new("Frame", switchBg)
    circle.Size = UDim2.fromOffset(16, 16)
    circle.Position = UDim2.new(0, 3, 0.5, -8)
    Instance.new("UICorner", circle).CornerRadius = UDim.new(1, 0)

    local function SetState(state)
        switchBg.BackgroundColor3 = state and Color3.fromRGB(240, 240, 250) or Color3.fromRGB(30, 30, 42)
        circle.Position = state and UDim2.new(1, -19, 0.5, -8) or UDim2.new(0, 3, 0.5, -8)
        circle.BackgroundColor3 = state and Color3.fromRGB(15, 15, 20) or Color3.fromRGB(200, 200, 210)
        label.TextColor3 = state and Text_Main or Text_Sub
    end
    
    btn.MouseButton1Click:Connect(function()
        Config[idKey] = not Config[idKey]
        SetState(Config[idKey])
        if not Config[idKey] then
            if idKey == "PlayerESPEnabled" then
                for _, plr in ipairs(Players:GetPlayers()) do if plr.Character then Disable3DESP(plr.Character) end end
            elseif idKey == "BotESPEnabled" then
                for _, bot in ipairs(TargetCache.Bots) do Disable3DESP(bot) end
            elseif idKey == "ItemBoxESPEnabled" then
                for _, obj in ipairs(TargetCache.ItemBoxes) do Disable3DESP(obj) end
            elseif idKey == "ExitESPEnabled" then
                for _, obj in ipairs(TargetCache.Exits) do Disable3DESP(obj) end
            elseif idKey == "CorpseESPEnabled" then
                for _, obj in ipairs(TargetCache.Corpses) do Disable3DESP(obj) end
            end
        end
        if callback then callback(Config[idKey]) end
    end)
    
    UI_Elements.Toggles[idKey] = SetState
    SetState(Config[idKey])
end

local function CreateModeButton(parent)
    local btn = Instance.new("TextButton", parent)
    btn.Size = UDim2.new(1, 0, 0, 44)
    btn.BackgroundColor3 = Card_BG
    btn.Text = "Aim Mode: " .. Config.AimMode
    btn.TextColor3 = Text_Main
    btn.Font = Enum.Font.GothamMedium
    btn.TextSize = Config.UITextSize
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 10)
    local stroke = Instance.new("UIStroke", btn)
    stroke.Thickness = 1
    stroke.Color = Stroke_Color
    table.insert(DynamicUITexts, btn)

    btn.MouseButton1Click:Connect(function()
        if Config.AimMode == "Hold RMB" then
            Config.AimMode = "Auto Lock (Always)"
        else
            Config.AimMode = "Hold RMB"
        end
        btn.Text = "Aim Mode: " .. Config.AimMode
        SaveConfig()
    end)
    
    UI_Elements.Buttons["AimModeBtn"] = btn
end

local function CreateSlider(title, minVal, maxVal, idKey, isFloat, parent, callback)
    local frame = Instance.new("Frame", parent)
    frame.Size = UDim2.new(1, 0, 0, 52)
    frame.BackgroundColor3 = Card_BG
    Instance.new("UICorner", frame).CornerRadius = UDim.new(0, 10)
    local stroke = Instance.new("UIStroke", frame)
    stroke.Thickness = 1
    stroke.Color = Stroke_Color

    local label = Instance.new("TextLabel", frame)
    label.Size = UDim2.new(1, -28, 0, 18)
    label.Position = UDim2.fromOffset(16, 8)
    label.BackgroundTransparency = 1
    label.TextColor3 = Text_Main
    label.TextSize = Config.UITextSize
    label.Font = Enum.Font.GothamBold
    label.TextXAlignment = Enum.TextXAlignment.Left
    table.insert(DynamicUITexts, label)

    local sliderBar = Instance.new("Frame", frame)
    sliderBar.Size = UDim2.new(1, -32, 0, 6)
    sliderBar.Position = UDim2.new(0, 16, 0, 34)
    sliderBar.BackgroundColor3 = Color3.fromRGB(30, 30, 42)
    Instance.new("UICorner", sliderBar).CornerRadius = UDim.new(1, 0)

    local sliderFill = Instance.new("Frame", sliderBar)
    sliderFill.BackgroundColor3 = Color3.fromRGB(240, 240, 250)
    Instance.new("UICorner", sliderFill).CornerRadius = UDim.new(1, 0)

    local function SetValue(val)
        val = math.clamp(val, minVal, maxVal)
        local pos = (val - minVal) / (maxVal - minVal)
        sliderFill.Size = UDim2.new(pos, 0, 1, 0)
        label.Text = isFloat and (title .. ": " .. string.format("%.2f", val)) or (title .. ": " .. math.floor(val))
    end

    local draggingSlider = false
    local function updateValue(input)
        local pos = math.clamp((input.Position.X - sliderBar.AbsolutePosition.X) / sliderBar.AbsoluteSize.X, 0, 1)
        local val = isFloat and tonumber(string.format("%.2f", minVal + ((maxVal - minVal) * pos))) or math.floor(minVal + ((maxVal - minVal) * pos))
        Config[idKey] = val
        SetValue(val)
        if callback then callback(val) end
    end

    sliderBar.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            draggingSlider = true
            updateValue(input)
        end
    end)
    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            draggingSlider = false
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if draggingSlider and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            updateValue(input)
        end
    end)

    UI_Elements.Sliders[idKey] = SetValue
    SetValue(Config[idKey])
end

-- สร้าง UI Elements ฝั่งซ้าย
CreateToggle("Aim Lock Enabled", "AimEnabled", LeftCol)
CreateModeButton(LeftCol)
CreateToggle("Bot Lock", "BotLockEnabled", LeftCol)
CreateToggle("Wall Check", "WallCheckEnabled", LeftCol)
CreateToggle("Prediction", "PredictionEnabled", LeftCol)
CreateSlider("Aim Smoothness", 0.00, 0.95, "AimSmoothness", true, LeftCol)
CreateSlider("FOV Radius", 50, 800, "FOV", false, LeftCol)
CreateToggle("Show FOV Circle", "ShowFOVCircle", LeftCol)

CreateToggle("No Recoil", "NoRecoilEnabled", LeftCol)
CreateToggle("Safe Fire Rate Booster", "FireRateEnabled", LeftCol)
CreateSlider("Fire Rate Boost", 1.0, 3.0, "FireRateMultiplier", true, LeftCol)

-- สร้าง UI Elements ฝั่งขวา
CreateToggle("ESP Player", "PlayerESPEnabled", RightCol)
CreateSlider("Player Distance", 100, 10000, "PlayerESPDistance", false, RightCol)

CreateToggle("Bot ESP Only", "BotESPEnabled", RightCol)
CreateSlider("Bot Distance", 100, 10000, "BotESPDistance", false, RightCol)

CreateToggle("Item & Box ESP", "ItemBoxESPEnabled", RightCol)
CreateSlider("Item Distance", 50, 5000, "ItemBoxESPDistance", false, RightCol)

CreateToggle("Safezone / Extract ESP", "ExitESPEnabled", RightCol)
CreateSlider("Safezone Distance", 100, 10000, "ExitESPDistance", false, RightCol)

CreateToggle("Corpse ESP", "CorpseESPEnabled", RightCol)
CreateSlider("Corpse Distance", 50, 10000, "CorpseESPDistance", false, RightCol)

-- สไลเดอร์ปรับขนาดตัวหนังสือ (ESP & UI Text Size)
CreateSlider("ESP Text Size", 8, 24, "ESPTextSize", false, RightCol, function(v)
    for _, data in pairs(ESPCache) do
        if data.TagText then
            data.TagText.TextSize = v
        end
    end
end)

CreateSlider("UI Text Size", 10, 18, "UITextSize", false, RightCol, function(v)
    UpdateAllUITextSizes(v)
end)

CreateToggle("Full Bright", "FullBrightEnabled", RightCol, function(v) 
    ApplyFullBright()
end)
CreateSlider("Full Bright Intensity", 1.0, 10.0, "FullBrightIntensity", true, RightCol, function(v)
    ApplyFullBright()
end)

local SaveBtn = Instance.new("TextButton", Footer)
SaveBtn.Size = UDim2.new(0.32, 0, 1, 0)
SaveBtn.BackgroundColor3 = Color3.fromRGB(240, 240, 250)
SaveBtn.Text = "Save Config"
SaveBtn.TextColor3 = Color3.fromRGB(15, 15, 20)
SaveBtn.Font = Enum.Font.GothamBold
SaveBtn.TextSize = 13
Instance.new("UICorner", SaveBtn).CornerRadius = UDim.new(0, 10)

local LoadBtn = Instance.new("TextButton", Footer)
LoadBtn.Size = UDim2.new(0.32, 0, 1, 0)
LoadBtn.Position = UDim2.new(0.34, 0, 0, 0)
LoadBtn.BackgroundColor3 = Card_BG
LoadBtn.Text = "Load Config"
LoadBtn.TextColor3 = Text_Main
LoadBtn.Font = Enum.Font.GothamBold
LoadBtn.TextSize = 13
Instance.new("UICorner", LoadBtn).CornerRadius = UDim.new(0, 10)

local DeleteBtn = Instance.new("TextButton", Footer)
DeleteBtn.Size = UDim2.new(0.32, 0, 1, 0)
DeleteBtn.Position = UDim2.new(0.68, 0, 0, 0)
DeleteBtn.BackgroundColor3 = Color3.fromRGB(45, 20, 25)
DeleteBtn.Text = "Delete Config"
DeleteBtn.TextColor3 = Color3.fromRGB(255, 100, 100)
DeleteBtn.Font = Enum.Font.GothamBold
DeleteBtn.TextSize = 13
Instance.new("UICorner", DeleteBtn).CornerRadius = UDim.new(0, 10)

local function RefreshUI()
    for k, setFunc in pairs(UI_Elements.Toggles) do setFunc(Config[k]) end
    for k, setFunc in pairs(UI_Elements.Sliders) do setFunc(Config[k]) end
    if UI_Elements.Buttons["AimModeBtn"] then
        UI_Elements.Buttons["AimModeBtn"].Text = "Aim Mode: " .. Config.AimMode
    end
    UpdateAllUITextSizes(Config.UITextSize)
    ApplyFullBright()
end

SaveBtn.MouseButton1Click:Connect(function()
    SaveConfig()
    SaveBtn.Text = "Saved!"
    task.wait(1)
    SaveBtn.Text = "Save Config"
end)

LoadBtn.MouseButton1Click:Connect(function()
    LoadConfig()
    RefreshUI()
    LoadBtn.Text = "Loaded!"
    task.wait(1)
    LoadBtn.Text = "Load Config"
end)

DeleteBtn.MouseButton1Click:Connect(function()
    DeleteConfig()
    RefreshUI()
    DeleteBtn.Text = "Deleted & Reset!"
    task.wait(1)
    DeleteBtn.Text = "Delete Config"
end)

RefreshUI()

-- Loop หลัก RenderStepped
table.insert(Connections, RunService.RenderStepped:Connect(function(dt)
    local camPos = Camera.CFrame.Position
    local mousePos = UserInputService:GetMouseLocation()
    
    FOVCircle.Size = UDim2.fromOffset(Config.FOV * 2, Config.FOV * 2)
    FOVCircle.Position = UDim2.fromOffset(mousePos.X, mousePos.Y)
    FOVCircle.Visible = Config.ShowFOVCircle

    if Config.FireRateEnabled and LocalPlayer.Character then
        local tool = LocalPlayer.Character:FindFirstChildOfClass("Tool")
        if tool then
            for _, v in ipairs(tool:GetDescendants()) do
                if v:IsA("NumberValue") or v:IsA("IntValue") then
                    local vName = v.Name:lower()
                    if vName:find("cooldown") or vName:find("delay") then
                        v.Value = math.max(0.01, v.Value / Config.FireRateMultiplier)
                    end
                end
            end
        end
    end

    local shouldAim = false
    if Config.AimEnabled then
        if Config.AimMode == "Auto Lock (Always)" then
            shouldAim = true
        elseif Config.AimMode == "Hold RMB" and isRMBDown then
            shouldAim = true
        end
    end

    if shouldAim then
        local targetPart = GetClosestTarget()
        if targetPart then
            local targetPos = targetPart.Position
            if Config.PredictionEnabled and targetPart.Parent and targetPart.Parent:FindFirstChild("HumanoidRootPart") then
                local vel = targetPart.Parent.HumanoidRootPart.AssemblyLinearVelocity
                targetPos = targetPos + (vel * 0.033)
            end

            local currentCF = Camera.CFrame
            local targetCF = CFrame.new(currentCF.Position, targetPos)
            
            local lerpFactor = math.clamp((1 - Config.AimSmoothness) * (dt * 60), 0.01, 1.0)
            Camera.CFrame = currentCF:Lerp(targetCF, lerpFactor)
        end
    end

    if Config.PlayerESPEnabled then
        for _, plr in ipairs(Players:GetPlayers()) do
            if plr ~= LocalPlayer and plr.Character then
                local char = plr.Character
                local mainPart = GetMainPart(char)
                local hum = char:FindFirstChildOfClass("Humanoid")
                
                if mainPart and hum and hum.Health > 0 then
                    local dist = math.floor((mainPart.Position - camPos).Magnitude)
                    if dist <= Config.PlayerESPDistance then
                        local tagText = Apply3DESP(char, Color3.fromRGB(255, 60, 60))
                        if tagText then
                            local heldItem = GetPlayerHeldItem(plr)
                            tagText.Text = string.format("%s [%dm]\nItems: %s", plr.DisplayName, dist, heldItem)
                        end
                    else
                        Disable3DESP(char)
                    end
                else
                    Disable3DESP(char)
                end
            end
        end
    end

    if Config.BotESPEnabled then
        for i = #TargetCache.Bots, 1, -1 do
            local bot = TargetCache.Bots[i]
            if bot and bot.Parent then
                local mainPart = GetMainPart(bot)
                local hum = bot:FindFirstChildOfClass("Humanoid")
                if mainPart and hum and hum.Health > 0 then
                    local dist = math.floor((mainPart.Position - camPos).Magnitude)
                    if dist <= Config.BotESPDistance then
                        local tagText = Apply3DESP(bot, Color3.fromRGB(255, 170, 0))
                        if tagText then
                            tagText.Text = string.format("[BOT] %s [%dm]", bot.Name, dist)
                        end
                    else
                        Disable3DESP(bot)
                    end
                else
                    Disable3DESP(bot)
                end
            else
                table.remove(TargetCache.Bots, i)
            end
        end
    end

    if Config.ItemBoxESPEnabled then
        for i = #TargetCache.ItemBoxes, 1, -1 do
            local obj = TargetCache.ItemBoxes[i]
            if obj and obj.Parent then
                local mainPart = GetMainPart(obj)
                if mainPart then
                    local dist = math.floor((mainPart.Position - camPos).Magnitude)
                    if dist <= Config.ItemBoxESPDistance then
                        local tagText = Apply3DESP(obj, Color3.fromRGB(0, 230, 255))
                        if tagText then
                            tagText.Text = string.format("%s [%dm]", GetItemName(obj) or obj.Name, dist)
                        end
                    else
                        Disable3DESP(obj)
                    end
                end
            else
                table.remove(TargetCache.ItemBoxes, i)
            end
        end
    end

    if Config.ExitESPEnabled then
        for i = #TargetCache.Exits, 1, -1 do
            local obj = TargetCache.Exits[i]
            if obj and obj.Parent then
                local mainPart = GetMainPart(obj)
                if mainPart then
                    local dist = math.floor((mainPart.Position - camPos).Magnitude)
                    if dist <= Config.ExitESPDistance then
                        local tagText = Apply3DESP(obj, Color3.fromRGB(50, 255, 100))
                        if tagText then
                            tagText.Text = string.format("[EXTRACT] %s [%dm]", obj.Name, dist)
                        end
                    else
                        Disable3DESP(obj)
                    end
                end
            else
                table.remove(TargetCache.Exits, i)
            end
        end
    end

    if Config.CorpseESPEnabled then
        for i = #TargetCache.Corpses, 1, -1 do
            local obj = TargetCache.Corpses[i]
            if obj and obj.Parent then
                local mainPart = GetMainPart(obj)
                if mainPart then
                    local dist = math.floor((mainPart.Position - camPos).Magnitude)
                    if dist <= Config.CorpseESPDistance then
                        local tagText = Apply3DESP(obj, Color3.fromRGB(150, 150, 150))
                        if tagText then
                            tagText.Text = string.format("[CORPSE] %s [%dm]", obj.Name, dist)
                        end
                    else
                        Disable3DESP(obj)
                    end
                end
            else
                table.remove(TargetCache.Corpses, i)
            end
        end
    end
end))
