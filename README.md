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

if ContainerParent:FindFirstChild("REDJOHN") then
    ContainerParent:FindFirstChild("REDJOHN"):Destroy()
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
    
    ESPTextSize = 13,
    UITextSize = 13,

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
local function SafeConnect(signal, func)
    local conn = signal:Connect(func)
    table.insert(Connections, conn)
    return conn
end

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
        AimKeybindButtonRef.Text = "Mode Key: " .. tostring(AimKeybindButtonRef.Text and tostring(AimModeKeybind.Name) or "")
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

SafeConnect(Lighting.Changed, function()
    if Config.FullBrightEnabled then
        ApplyFullBright()
    end
end)

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "REDJOHN"
ScreenGui.ResetOnSpawn = false
ScreenGui.Parent = ContainerParent

local FOVCircle = Instance.new("Frame", ScreenGui)
FOVCircle.Name = "FOVCircle"
FOVCircle.BackgroundTransparency = 1
FOVCircle.AnchorPoint = Vector2.new(0.5, 0.5)
FOVCircle.Size = UDim2.fromOffset(Config.FOV * 2, Config.FOV * 2)
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

local function GetBotName(botModel)
    if not botModel then return "Bot" end
    if botModel:FindFirstChild("DisplayName") and botModel.DisplayName:IsA("StringValue") then
        return botModel.DisplayName.Value
    elseif botModel:GetAttribute("DisplayName") then
        return tostring(botModel:GetAttribute("DisplayName"))
    elseif botModel:GetAttribute("BotName") then
        return tostring(botModel:GetAttribute("BotName"))
    elseif botModel:FindFirstChild("Username") and botModel.Username:IsA("StringValue") then
        return botModel.Username.Value
    end
    return botModel.Name
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
            addItem(GetItemName(item) or item.Name)
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
    end

    return #itemsFound > 0 and table.concat(itemsFound, ", ") or "Nothing"
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
        local tagText, bb, healthFrame, healthFill, healthText

        if mainPart then
            bb = Instance.new("BillboardGui")
            bb.Name = "PEERANAT_3DTag"
            bb.Size = UDim2.fromOffset(300, 90)
            bb.StudsOffset = Vector3.new(0, 3.2, 0)
            bb.AlwaysOnTop = true

            tagText = Instance.new("TextLabel", bb)
            tagText.Name = "TagText"
            tagText.Size = UDim2.new(1, 0, 0, 48)
            tagText.Position = UDim2.new(0, 0, 0, 0)
            tagText.BackgroundTransparency = 1
            tagText.TextStrokeTransparency = 0.2
            tagText.TextWrapped = true
            tagText.TextSize = Config.ESPTextSize
            tagText.Font = Enum.Font.GothamBold

            healthFrame = Instance.new("Frame", bb)
            healthFrame.Name = "HealthFrame"
            healthFrame.Size = UDim2.new(0.6, 0, 0, 10)
            healthFrame.Position = UDim2.new(0.2, 0, 0, 52)
            healthFrame.BackgroundColor3 = Color3.fromRGB(20, 25, 20)
            healthFrame.BorderSizePixel = 0
            healthFrame.Visible = false

            local hStroke = Instance.new("UIStroke", healthFrame)
            hStroke.Color = Color3.fromRGB(15, 15, 15)
            hStroke.Thickness = 1

            healthFill = Instance.new("Frame", healthFrame)
            healthFill.Name = "HealthFill"
            healthFill.Size = UDim2.new(1, 0, 1, 0)
            healthFill.BackgroundColor3 = Color3.fromRGB(135, 185, 80)
            healthFill.BorderSizePixel = 0

            healthText = Instance.new("TextLabel", healthFrame)
            healthText.Name = "HealthText"
            healthText.Size = UDim2.new(1, 0, 1, 0)
            healthText.Position = UDim2.new(0, 0, 0, 0)
            healthText.BackgroundTransparency = 1
            healthText.TextColor3 = Color3.fromRGB(255, 255, 255)
            healthText.TextStrokeTransparency = 0.3
            healthText.TextSize = 9
            healthText.Font = Enum.Font.GothamBold

            bb.Parent = mainPart
        end

        data = { Highlight = highlight, Billboard = bb, TagText = tagText, HealthFrame = healthFrame, HealthFill = healthFill, HealthText = healthText }
        ESPCache[model] = data
        
        SafeConnect(model.Destroying, function()
            ESPCache[model] = nil
        end)
    end

    data.Highlight.Enabled = true
    data.Highlight.FillColor = color
    data.Highlight.OutlineColor = color

    if data.Billboard then
        data.Billboard.Enabled = true
        data.TagText.TextColor3 = color
        data.TagText.TextSize = Config.ESPTextSize
    end

    return data
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
SafeConnect(workspace.DescendantAdded, ClassifyAndAddObject)

local function setupPlayerRespawnESP(plr)
    SafeConnect(plr.CharacterAdded, function(char)
        task.wait(0.3)
        if ESPCache[char] then
            Disable3DESP(char)
            ESPCache[char] = nil
        end
    end)
end

for _, plr in ipairs(Players:GetPlayers()) do
    setupPlayerRespawnESP(plr)
end
SafeConnect(Players.PlayerAdded, setupPlayerRespawnESP)

SafeConnect(UserInputService.InputBegan, function(input, gameProcessed)
    if input.UserInputType == Enum.UserInputType.MouseButton2 then
        isRMBDown = true
    end

    if isListeningForKey then
        if input.UserInputType == Enum.UserInputType.Keyboard then
            ToggleKeybind = input.KeyCode
            isListeningForKey = false
            if KeybindButtonRef then KeybindButtonRef.Text = "UI Key: " .. tostring(ToggleKeybind.Name) end
            SaveConfig()
        end
        return
    end

    if isListeningForAimKey then
        if input.UserInputType == Enum.UserInputType.Keyboard then
            AimModeKeybind = input.KeyCode
            isListeningForAimKey = false
            if AimKeybindButtonRef then AimKeybindButtonRef.Text = "Mode Key: " .. tostring(AimModeKeybind.Name) end
            SaveConfig()
        end
        return
    end

    if input.KeyCode == ToggleKeybind then
        ScreenGui.Enabled = not ScreenGui.Enabled
    elseif input.KeyCode == AimModeKeybind then
        Config.AimMode = Config.AimMode == "Hold RMB" and "Auto Lock (Always)" or "Hold RMB"
        if UI_Elements.Buttons["AimModeBtn"] then UI_Elements.Buttons["AimModeBtn"].Text = "Aim Mode: " .. Config.AimMode end
        SaveConfig()
    end
end)

SafeConnect(UserInputService.InputEnded, function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton2 then
        isRMBDown = false
    end
end)

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
Title.Text = "REDJOHN"
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
        if conn and typeof(conn) == "RBXScriptConnection" and conn.Connected then
            pcall(function() conn:Disconnect() end)
        end
    end
    table.clear(Connections)
    DestroyAllESP()
    
    pcall(function()
        if ScreenGui then ScreenGui:Destroy() end
    end)
end

Close.MouseButton1Click:Connect(function()
    Main.Visible = false
    UnloadScript()
end)

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
        Config.AimMode = Config.AimMode == "Hold RMB" and "Auto Lock (Always)" or "Hold RMB"
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
    sliderBar.InputChanged:Connect(function(input)
        if draggingSlider and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            updateValue(input)
        end
    end)

    UI_Elements.Sliders[idKey] = SetValue
    SetValue(Config[idKey])
end

local function CreateButton(text, callback, parent, colorOverride)
    local btn = Instance.new("TextButton", parent)
    btn.Size = UDim2.new(1, 0, 0, 44)
    btn.BackgroundColor3 = colorOverride or Card_BG
    btn.Text = text
    btn.TextColor3 = Text_Main
    btn.Font = Enum.Font.GothamMedium
    btn.TextSize = Config.UITextSize
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 10)
    local stroke = Instance.new("UIStroke", btn)
    stroke.Thickness = 1
    stroke.Color = Stroke_Color
    table.insert(DynamicUITexts, btn)

    btn.MouseButton1Click:Connect(function()
        if callback then callback() end
    end)
    return btn
end

-- Populate UI Left/Right
CreateToggle("Aim Enabled", "AimEnabled", LeftCol)
CreateModeButton(LeftCol)
CreateToggle("Bot Lock Enabled", "BotLockEnabled", LeftCol)
CreateToggle("Wall Check Enabled", "WallCheckEnabled", LeftCol)
CreateToggle("Prediction Enabled", "PredictionEnabled", LeftCol)
CreateSlider("Aim Smoothness", 0.05, 1.0, "AimSmoothness", true, LeftCol)
CreateSlider("FOV Radius", 50, 600, "FOV", false, LeftCol, function(val)
    FOVCircle.Size = UDim2.fromOffset(val * 2, val * 2)
end)
CreateToggle("Show FOV Circle", "ShowFOVCircle", LeftCol, function(state)
    FOVCircle.Visible = state
end)
CreateToggle("No Recoil", "NoRecoilEnabled", LeftCol)

CreateToggle("Player ESP", "PlayerESPEnabled", RightCol)
CreateSlider("Player Distance", 100, 10000, "PlayerESPDistance", false, RightCol)
CreateToggle("Bot ESP", "BotESPEnabled", RightCol)
CreateSlider("Bot Distance", 100, 10000, "BotESPDistance", false, RightCol)
CreateToggle("Item Box ESP", "ItemBoxESPEnabled", RightCol)
CreateSlider("Item Distance", 100, 10000, "ItemBoxESPDistance", false, RightCol)
CreateToggle("Exit ESP", "ExitESPEnabled", RightCol)
CreateSlider("Exit Distance", 100, 10000, "ExitESPDistance", false, RightCol)
CreateToggle("Corpse ESP", "CorpseESPEnabled", RightCol)
CreateSlider("Corpse Distance", 100, 10000, "CorpseESPDistance", false, RightCol)

CreateToggle("Fullbright", "FullBrightEnabled", RightCol, ApplyFullBright)
CreateSlider("Fullbright Intensity", 1.0, 10.0, "FullBrightIntensity", true, RightCol, function()
    if Config.FullBrightEnabled then ApplyFullBright() end
end)

CreateSlider("ESP Text Size", 9, 24, "ESPTextSize", false, RightCol, function(val) Config.ESPTextSize = val end)
CreateSlider("UI Text Size", 10, 20, "UITextSize", false, RightCol, function(val)
    Config.UITextSize = val
    UpdateAllUITextSizes(val)
end)

CreateButton("Save Config", SaveConfig, RightCol)
CreateButton("Reset / Delete Config", DeleteConfig, RightCol, Color3.fromRGB(45, 20, 25))

local FooterText = Instance.new("TextLabel", Footer)
FooterText.Size = UDim2.new(1, 0, 1, 0)
FooterText.BackgroundTransparency = 1
FooterText.Text = "REDJOHN HUB | Loaded Successfully"
FooterText.TextColor3 = Text_Sub
FooterText.TextSize = 12
FooterText.Font = Enum.Font.GothamMedium

-- Optimized Main Loop
local cleanTick = 0
local function CleanInvalidCache(tbl)
    for i = #tbl, 1, -1 do
        local obj = tbl[i]
        if not obj or not obj.Parent or (obj:IsA("Model") and not obj:FindFirstChildOfClass("Humanoid") and not obj.PrimaryPart) then
            if ESPCache[obj] then
                Disable3DESP(obj)
                ESPCache[obj] = nil
            end
            table.remove(tbl, i)
        end
    end
end

SafeConnect(RunService.RenderStepped, function(deltaTime)
    local camPos = Camera.CFrame.Position
    local mouseLoc = UserInputService:GetMouseLocation()
    deltaTime = math.clamp(deltaTime, 0.001, 0.1)

    local camSize = Camera.ViewportSize
    FOVCircle.Position = UDim2.fromOffset(camSize.X / 2, camSize.Y / 2)
    if Config.FullBrightEnabled then
        Lighting.Brightness = Config.FullBrightIntensity
    end

    cleanTick += deltaTime
    if cleanTick >= 1.0 then
        cleanTick = 0
        CleanInvalidCache(TargetCache.Bots)
        CleanInvalidCache(TargetCache.ItemBoxes)
        CleanInvalidCache(TargetCache.Exits)
        CleanInvalidCache(TargetCache.Corpses)
    end

    local function processESP(list, enabled, distLimit, color, labelPrefix, showHealth)
        if not enabled then
            for _, obj in ipairs(list) do Disable3DESP(obj) end
            return
        end
        for _, obj in ipairs(list) do
            if obj and obj.Parent then
                local mainPart = GetMainPart(obj)
                if mainPart then
                    local dist = (mainPart.Position - camPos).Magnitude
                    if dist <= distLimit then
                        local data = Apply3DESP(obj, color)
                        if data and data.TagText then
                            local textVal = labelPrefix
                            if labelPrefix == "Bot" then
                                textVal = GetBotName(obj)
                            elseif labelPrefix == "Item" then
                                textVal = GetItemName(obj) or "Item"
                            end
                            data.TagText.Text = string.format("%s\n%dm", textVal, math.floor(dist))
                            
                            if showHealth then
                                local hum = obj:FindFirstChildOfClass("Humanoid")
                                if hum and data.HealthFrame then
                                    data.HealthFrame.Visible = true
                                    local hp = math.clamp(hum.Health, 0, hum.MaxHealth)
                                    data.HealthFill.Size = UDim2.new(hp / hum.MaxHealth, 0, 1, 0)
                                    data.HealthText.Text = math.floor(hp) .. "HP"
                                end
                            elseif data.HealthFrame then
                                data.HealthFrame.Visible = false
                            end
                        end
                    else
                        Disable3DESP(obj)
                    end
                end
            end
        end
    end

    if Config.PlayerESPEnabled then
        for _, plr in ipairs(Players:GetPlayers()) do
            if plr ~= LocalPlayer and plr.Character then
                local mainPart = GetMainPart(plr.Character)
                if mainPart then
                    local dist = (mainPart.Position - camPos).Magnitude
                    if dist <= Config.PlayerESPDistance then
                        local data = Apply3DESP(plr.Character, Color3.fromRGB(255, 80, 80))
                        if data and data.TagText then
                            local heldItem = GetPlayerHeldItem(plr)
                            local hum = plr.Character:FindFirstChildOfClass("Humanoid")
                            local hp = hum and math.floor(hum.Health) or 100
                            data.TagText.Text = string.format("%s\nDist: %dm\nHeld: %s", plr.Name, math.floor(dist), heldItem)
                            if data.HealthFrame then
                                data.HealthFrame.Visible = true
                                data.HealthFill.Size = UDim2.new(math.clamp(hp / 100, 0, 1), 0, 1, 0)
                                data.HealthText.Text = hp .. "HP"
                            end
                        end
                    else
                        Disable3DESP(plr.Character)
                    end
                end
            end
        end
    else
        for _, plr in ipairs(Players:GetPlayers()) do
            if plr.Character then Disable3DESP(plr.Character) end
        end
    end

    processESP(TargetCache.Bots, Config.BotESPEnabled, Config.BotESPDistance, Color3.fromRGB(255, 165, 0), "Bot", true)
    processESP(TargetCache.ItemBoxes, Config.ItemBoxESPEnabled, Config.ItemBoxESPDistance, Color3.fromRGB(80, 255, 120), "Item", false)
    processESP(TargetCache.Exits, Config.ExitESPEnabled, Config.ExitESPDistance, Color3.fromRGB(80, 200, 255), "Extract", false)
    processESP(TargetCache.Corpses, Config.CorpseESPEnabled, Config.CorpseESPDistance, Color3.fromRGB(160, 160, 160), "Corpse", false)

    if Config.AimEnabled and (Config.AimMode == "Auto Lock (Always)" or (Config.AimMode == "Hold RMB" and isRMBDown)) then
        local targetPart = nil
        local shortestDist = Config.FOV

        local function checkTargetCandidate(model)
            if not model or model == LocalPlayer.Character then return end
            local hum = model:FindFirstChildOfClass("Humanoid")
            if hum and hum.Health <= 0 then return end
            
            local part = model:FindFirstChild("Head") or GetMainPart(model)
            if part then
                if Config.WallCheckEnabled then
                    local origin = camPos
                    local direction = (part.Position - origin)
                    local rayParams = RaycastParams.new()
                    rayParams.FilterType = Enum.RaycastFilterType.Blacklist
                    local filterList = {LocalPlayer.Character}
                    if ScreenGui and ScreenGui.Parent then table.insert(filterList, ScreenGui) end
                    rayParams.FilterDescendantsInstances = filterList
                    local result = workspace:Raycast(origin, direction, rayParams)
                    if result and result.Instance and not result.Instance:IsDescendantOf(model) then
                        return
                    end
                end

                local screenPos, onScreen = Camera:WorldToViewportPoint(part.Position)
                if onScreen then
                    local mag = (Vector2.new(screenPos.X, screenPos.Y) - mouseLoc).Magnitude
                    if mag < shortestDist then
                        shortestDist = mag
                        targetPart = part
                    end
                end
            end
        end

        for _, plr in ipairs(Players:GetPlayers()) do
            if plr ~= LocalPlayer and plr.Character then
                checkTargetCandidate(plr.Character)
            end
        end
        if Config.BotLockEnabled then
            for _, bot in ipairs(TargetCache.Bots) do
                checkTargetCandidate(bot)
            end
        end

        if targetPart then
            local predPos = targetPart.Position
            if Config.PredictionEnabled and targetPart.AssemblyLinearVelocity then
                predPos = predPos + (targetPart.AssemblyLinearVelocity * 0.08)
            end

            local goalCFrame = CFrame.new(Camera.CFrame.Position, predPos)
            local smoothness = math.clamp(Config.AimSmoothness, 0.02, 1)
            Camera.CFrame = Camera.CFrame:Lerp(goalCFrame, 1 - math.pow(smoothness, 0.5))
        end
    end
end)
