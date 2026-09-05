local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local CoreGui = game:GetService("CoreGui")
local Lighting = game:GetService("Lighting")
local HttpService = game:GetService("HttpService")

local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera

local ToggleKeybind = Enum.KeyCode.RightShift
local isListeningForKey = false

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
    BotLockEnabled = true,
    WallCheckEnabled = false,
    PredictionEnabled = true,
    AutoShootEnabled = false,
    AutoHoldShootEnabled = true, -- เปิดใช้งานคลิกขวาค้างยิงออโต้เป็นค่าเริ่มต้น
    AimOnRightClick = true,
    NoRecoilEnabled = true,
    FireRateEnabled = true,
    FireRateMultiplier = 10.0,
    ToggleKeybindName = "RightShift",
    
    PlayerESPEnabled = true,
    BotESPEnabled = true,
    ItemBoxESPEnabled = true,
    ExitESPEnabled = true,
    CorpseESPEnabled = true,
    FullBrightEnabled = false,
    ShowFOVCircle = true,
    
    FOV = 800,
    Smoothness = 0.0,
    PlayerESPDistance = 10000,
    BotESPDistance = 10000,
    ItemBoxESPDistance = 10000,
    ExitESPDistance = 10000,
    CorpseESPDistance = 10000
}

local Config = {}
for k, v in pairs(DefaultConfig) do Config[k] = v end

local ConfigFileName = "REDJOHN_HUB_Config.json"
local IsMinimized = false
local UI_Elements = { Toggles = {}, Sliders = {} }
local KeybindButtonRef = nil

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

local function SaveConfig()
    pcall(function()
        if writefile then
            Config.ToggleKeybindName = ToggleKeybind.Name
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
                            KeybindButtonRef.Text = "Key: " .. tostring(ToggleKeybind.Name)
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
    for k, v in pairs(DefaultConfig) do Config[k] = v end
    ToggleKeybind = Enum.KeyCode.RightShift
    if KeybindButtonRef then
        KeybindButtonRef.Text = "Key: " .. tostring(ToggleKeybind.Name)
    end
end

LoadConfig()

local function ApplyFullBright(enabled)
    pcall(function()
        if enabled then
            Lighting.Brightness = 2
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

Lighting.Changed:Connect(function()
    if Config.FullBrightEnabled then
        ApplyFullBright(true)
    end
end)

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
    if not obj then return "Unknown" end
    if obj:FindFirstChild("ItemName") and obj.ItemName:IsA("StringValue") then
        return obj.ItemName.Value
    elseif obj:GetAttribute("ItemName") then
        return tostring(obj:GetAttribute("ItemName"))
    elseif obj:GetAttribute("Name") then
        return tostring(obj:GetAttribute("Name"))
    end
    
    local tool = obj:FindFirstChildOfClass("Tool")
    if tool then
        return tool.Name
    end
    return obj.Name
end

local function Apply3DESP(model, color)
    if not model then return nil end
    
    local highlight = model:FindFirstChild("REDJOHN_3DHighlight")
    if not highlight then
        highlight = Instance.new("Highlight")
        highlight.Name = "REDJOHN_3DHighlight"
        highlight.Adornee = model
        highlight.FillTransparency = 0.6
        highlight.OutlineTransparency = 0.1
        highlight.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
        highlight.Parent = model
    end
    highlight.Enabled = true
    highlight.FillColor = color
    highlight.OutlineColor = color

    local mainPart = GetMainPart(model)
    if mainPart then
        local bb = mainPart:FindFirstChild("REDJOHN_3DTag")
        if not bb then
            bb = Instance.new("BillboardGui")
            bb.Name = "REDJOHN_3DTag"
            bb.Size = UDim2.fromOffset(150, 30)
            bb.StudsOffset = Vector3.new(0, 2.5, 0)
            bb.AlwaysOnTop = true

            local txt = Instance.new("TextLabel", bb)
            txt.Name = "TagText"
            txt.Size = UDim2.new(1, 0, 1, 0)
            txt.BackgroundTransparency = 1
            txt.TextStrokeTransparency = 0.4
            txt.TextSize = 13
            txt.Font = Enum.Font.GothamBold
            bb.Parent = mainPart
        end
        bb.Enabled = true
        if bb and bb:FindFirstChild("TagText") then
            bb.TagText.TextColor3 = color
            return bb.TagText
        end
    end
    return nil
end

local function Disable3DESP(model)
    if not model then return end
    local highlight = model:FindFirstChild("REDJOHN_3DHighlight")
    if highlight then highlight.Enabled = false end
    local mainPart = GetMainPart(model)
    if mainPart then
        local bb = mainPart:FindFirstChild("REDJOHN_3DTag")
        if bb then bb.Enabled = false end
    end
end

local function Remove3DESP(model)
    if not model then return end
    if model:FindFirstChild("REDJOHN_3DHighlight") then 
        model.REDJOHN_3DHighlight:Destroy() 
    end
    local mainPart = GetMainPart(model)
    if mainPart and mainPart:FindFirstChild("REDJOHN_3DTag") then 
        mainPart.REDJOHN_3DTag:Destroy() 
    end
end

local function CleanAllESP()
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr.Character then Remove3DESP(plr.Character) end
    end
    for _, bot in ipairs(TargetCache.Bots) do Remove3DESP(bot) end
    for _, obj in ipairs(TargetCache.ItemBoxes) do Remove3DESP(obj) end
    for _, obj in ipairs(TargetCache.Exits) do Remove3DESP(obj) end
    for _, obj in ipairs(TargetCache.Corpses) do Remove3DESP(obj) end
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

    if (obj:IsA("Model") or obj:IsA("BasePart") or obj:IsA("Tool")) then
        local hum = obj:FindFirstChildOfClass("Humanoid")
        local plr = Players:GetPlayerFromCharacter(obj)

        if hum and not plr then
            if hum.Health > 0 then
                table.insert(TargetCache.Bots, obj)
            else
                table.insert(TargetCache.Corpses, obj)
            end
        elseif not hum and not plr then
            if (name:find("bot") or name:find("npc") or name:find("enemy") or name:find("mob") or name:find("zombie") or name:find("guard")) and not isDoor then
                table.insert(TargetCache.Bots, obj)
            elseif obj:IsA("Tool") or name:find("item") or name:find("drop") or name:find("loot") or name:find("pickup") 
                or name:find("box") or name:find("crate") or name:find("chest") or name:find("container") or name:find("pack") or name:find("ammo") or name:find("weapon") then
                table.insert(TargetCache.ItemBoxes, obj)
            elseif not isDoor and (name:find("safezone") or name:find("safe_zone") or name:find("evac") or name:find("extract") or name:find("return_base") or name:find("spawnzone")) then
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

workspace.DescendantAdded:Connect(function(child)
    ClassifyAndAddObject(child)
end)

local IsRightMouseDown = false

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if input.UserInputType == Enum.UserInputType.MouseButton2 then
        IsRightMouseDown = true
    end
    
    if isListeningForKey then
        if input.UserInputType == Enum.UserInputType.Keyboard then
            ToggleKeybind = input.KeyCode
            isListeningForKey = false
            if KeybindButtonRef then
                KeybindButtonRef.Text = "Key: " .. tostring(ToggleKeybind.Name)
            end
            SaveConfig()
        end
        return
    end

    if input.KeyCode == ToggleKeybind then
        ScreenGui.Enabled = not ScreenGui.Enabled
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton2 then
        IsRightMouseDown = false
    end
end)

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
    if Config.AimOnRightClick and not IsRightMouseDown then
        return nil
    end

    local closestTarget = nil
    local shortestDistance = Config.FOV
    local mousePos = UserInputService:GetMouseLocation()

    if Config.BotLockEnabled then
        for _, bot in ipairs(TargetCache.Bots) do
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

local HeaderImage = Instance.new("ImageLabel", Header)
HeaderImage.Size = UDim2.new(1, 0, 1, 0)
HeaderImage.Position = UDim2.new(0, 0, 0, 0)
HeaderImage.BackgroundTransparency = 1
HeaderImage.Image = "rbxassetid://YOUR_IMAGE_ID"
HeaderImage.ScaleType = Enum.ScaleType.Crop
HeaderImage.ImageTransparency = 0.45

local HeaderGradient = Instance.new("UIGradient", Header)
HeaderGradient.Transparency = NumberSequence.new({
    NumberSequenceKeypoint.new(0, 0.2),
    NumberSequenceKeypoint.new(1, 0.8)
})

local Title = Instance.new("TextLabel", Header)
Title.Size = UDim2.new(1, -260, 1, 0)
Title.Position = UDim2.fromOffset(24, 0)
Title.BackgroundTransparency = 1
Title.Text = "REDJOHN_HUB (Hold Right-Click Auto Shoot)"
Title.TextColor3 = Text_Main
Title.TextSize = 17
Title.Font = Enum.Font.GothamBold
Title.TextXAlignment = Enum.TextXAlignment.Left

local KeybindButton = Instance.new("TextButton", Header)
KeybindButton.Size = UDim2.fromOffset(110, 36)
KeybindButton.Position = UDim2.new(1, -200, 0.5, -18)
KeybindButton.BackgroundColor3 = Card_BG
KeybindButton.Text = "Key: " .. tostring(ToggleKeybind.Name)
KeybindButton.TextColor3 = Text_Main
KeybindButton.TextSize = 11
KeybindButton.Font = Enum.Font.GothamMedium
Instance.new("UICorner", KeybindButton).CornerRadius = UDim.new(0, 10)
local keyStroke = Instance.new("UIStroke", KeybindButton)
keyStroke.Color = Stroke_Color
KeybindButtonRef = KeybindButton

KeybindButton.MouseButton1Click:Connect(function()
    isListeningForKey = true
    KeybindButton.Text = "Press Key..."
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
local minStroke = Instance.new("UIStroke", Minimize)
minStroke.Color = Stroke_Color

local Close = Instance.new("TextButton", Header)
Close.Size = UDim2.fromOffset(36, 36)
Close.Position = UDim2.new(1, -38, 0.5, -18)
Close.BackgroundColor3 = Color3.fromRGB(45, 20, 25)
Close.Text = "x"
Close.TextColor3 = Color3.fromRGB(255, 100, 100)
Close.TextSize = 14
Close.Font = Enum.Font.GothamBold
Instance.new("UICorner", Close).CornerRadius = UDim.new(0, 10)
local closeStroke = Instance.new("UIStroke", Close)
closeStroke.Color = Color3.fromRGB(80, 30, 35)

local ContentContainer = Instance.new("ScrollingFrame", Main)
ContentContainer.Size = UDim2.new(1, -32, 1, -145)
ContentContainer.Position = UDim2.fromOffset(16, 90)
ContentContainer.BackgroundTransparency = 1
ContentContainer.CanvasSize = UDim2.new(0, 0, 0, 0)
ContentContainer.AutomaticCanvasSize = Enum.AutomaticSize.Y
ContentContainer.ScrollBarThickness = 4
ContentContainer.ScrollBarImageColor3 = Color3.fromRGB(70, 70, 90)

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
    label.TextSize = 13
    label.TextXAlignment = Enum.TextXAlignment.Left

    local switchBg = Instance.new("Frame", btn)
    switchBg.Size = UDim2.fromOffset(42, 22)
    switchBg.Position = UDim2.new(1, -54, 0.5, -11)
    Instance.new("UICorner", switchBg).CornerRadius = UDim.new(1, 0)

    local circle = Instance.new("Frame", switchBg)
    circle.Size = UDim2.fromOffset(16, 16)
    circle.Position = UDim2.new(0, 3, 0.5, -8)
    Instance.new("UICorner", circle).CornerRadius = UDim.new(1, 0)
    circle.BackgroundColor3 = Color3.fromRGB(255, 255, 255)

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
                for _, plr in ipairs(Players:GetPlayers()) do if plr.Character then Remove3DESP(plr.Character) end end
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
    label.TextSize = 13
    label.Font = Enum.Font.GothamBold
    label.TextXAlignment = Enum.TextXAlignment.Left

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
        label.Text = isFloat and (title .. ": " .. string.format("%.1fx", val)) or (title .. ": " .. math.floor(val))
    end

    local draggingSlider = false
    local function updateValue(input)
        local pos = math.clamp((input.Position.X - sliderBar.AbsolutePosition.X) / sliderBar.AbsoluteSize.X, 0, 1)
        local val = isFloat and tonumber(string.format("%.1f", minVal + ((maxVal - minVal) * pos))) or math.floor(minVal + ((maxVal - minVal) * pos))
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

CreateToggle("Aimlock (Player 100%)", "AimEnabled", LeftCol)
CreateToggle("Bot Headlock (100%)", "BotLockEnabled", LeftCol)
CreateToggle("Aim Only on Right Click", "AimOnRightClick", LeftCol)
CreateToggle("Hold Right-Click Auto Shoot", "AutoHoldShootEnabled", LeftCol) -- ฟังก์ชันใหม่ คลิกขวาค้างยิงออโต้
CreateToggle("Wall Check (Off = Through Walls)", "WallCheckEnabled", LeftCol)
CreateToggle("Prediction", "PredictionEnabled", LeftCol)
CreateToggle("Auto Shoot (General)", "AutoShootEnabled", LeftCol)
CreateToggle("No Recoil", "NoRecoilEnabled", LeftCol)
CreateToggle("Fire Rate Booster (Ultra)", "FireRateEnabled", LeftCol)
CreateSlider("Fire Rate Multiplier", 1.0, 10.0, "FireRateMultiplier", true, LeftCol)
CreateToggle("Show FOV Circle", "ShowFOVCircle", LeftCol)
CreateSlider("FOV Radius", 50, 800, "FOV", false, LeftCol)
CreateSlider("Aim Smoothness (0 = Instant)", 0.0, 0.1, "Smoothness", true, LeftCol)

CreateToggle("Player 3D ESP", "PlayerESPEnabled", RightCol)
CreateSlider("Player Distance", 100, 10000, "PlayerESPDistance", false, RightCol)

CreateToggle("Bot 3D ESP", "BotESPEnabled", RightCol)
CreateSlider("Bot Distance", 100, 10000, "BotESPDistance", false, RightCol)

CreateToggle("Item & Box ESP", "ItemBoxESPEnabled", RightCol)
CreateSlider("Item & Box Distance", 50, 3000, "ItemBoxESPDistance", false, RightCol)

CreateToggle("Safezone / Extract ESP", "ExitESPEnabled", RightCol)
CreateSlider("Safezone Distance", 100, 10000, "ExitESPDistance", false, RightCol)

CreateToggle("Corpse ESP", "CorpseESPEnabled", RightCol)
CreateSlider("Corpse Distance", 50, 10000, "CorpseESPDistance", false, RightCol)

CreateToggle("Full Bright", "FullBrightEnabled", RightCol, function(v) 
    ApplyFullBright(v)
end)

local Footer = Instance.new("Frame", Main)
Footer.Size = UDim2.new(1, -32, 0, 45)
Footer.Position = UDim2.new(0, 16, 1, -55)
Footer.BackgroundTransparency = 1

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
local loadStroke = Instance.new("UIStroke", LoadBtn)
loadStroke.Color = Stroke_Color

local DeleteBtn = Instance.new("TextButton", Footer)
DeleteBtn.Size = UDim2.new(0.32, 0, 1, 0)
DeleteBtn.Position = UDim2.new(0.68, 0, 0, 0)
DeleteBtn.BackgroundColor3 = Color3.fromRGB(45, 20, 25)
DeleteBtn.Text = "Delete Config"
DeleteBtn.TextColor3 = Color3.fromRGB(255, 100, 100)
DeleteBtn.Font = Enum.Font.GothamBold
DeleteBtn.TextSize = 13
Instance.new("UICorner", DeleteBtn).CornerRadius = UDim.new(0, 10)
local delStroke = Instance.new("UIStroke", DeleteBtn)
delStroke.Color = Color3.fromRGB(80, 30, 35)

local function RefreshUI()
    for k, setFunc in pairs(UI_Elements.Toggles) do setFunc(Config[k]) end
    for k, setFunc in pairs(UI_Elements.Sliders) do setFunc(Config[k]) end
    ApplyFullBright(Config.FullBrightEnabled)
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

task.spawn(function()
    pcall(function()
        local mt = getrawmetatable(game)
        setreadonly(mt, false)
        local oldIndex = mt.__index
        
        mt.__index = newcclosure(function(self, k)
            if Config.NoRecoilEnabled and (k == "Recoil" or k == "CameraRecoil" or k == "Spread" or k == "Kickback") then
                return 0
            end
            return oldIndex(self, k)
        end)
        setreadonly(mt, true)
    end)
end)

local IsRunning = true
local Connection

Connection = RunService.RenderStepped:Connect(function()
    if not IsRunning then
        Connection:Disconnect()
        return
    end

    local camPos = Camera.CFrame.Position
    local mousePos = UserInputService:GetMouseLocation()
    
    FOVCircle.Size = UDim2.fromOffset(Config.FOV * 2, Config.FOV * 2)
    FOVCircle.Position = UDim2.fromOffset(mousePos.X, mousePos.Y)
    FOVCircle.Visible = Config.ShowFOVCircle

    if Config.NoRecoilEnabled and LocalPlayer.Character then
        local tool = LocalPlayer.Character:FindFirstChildOfClass("Tool")
        if tool then
            for _, v in ipairs(tool:GetDescendants()) do
                if v:IsA("NumberValue") or v:IsA("IntValue") then
                    local vName = v.Name:lower()
                    if vName:find("recoil") or vName:find("spread") or vName:find("kick") then
                        v.Value = 0
                    end
                end
            end
        end
    end

    if Config.FireRateEnabled and LocalPlayer.Character then
        local tool = LocalPlayer.Character:FindFirstChildOfClass("Tool")
        if tool then
            for _, v in ipairs(tool:GetDescendants()) do
                if v:IsA("NumberValue") or v:IsA("IntValue") then
                    local vName = v.Name:lower()
                    if vName:find("firerate") or vName:find("cooldown") or vName:find("delay") or vName:find("rpm") or vName:find("fire") or vName:find("rate") then
                        if v.Value > 0 then
                            if vName:find("rpm") or (vName:find("rate") and not vName:find("cooldown")) then
                                v.Value = v.Value * Config.FireRateMultiplier
                            else
                                v.Value = math.max(0.001, v.Value / Config.FireRateMultiplier)
                            end
                        end
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
                    local dist = (camPos - mainPart.Position).Magnitude
                    if dist <= Config.PlayerESPDistance then
                        local tag = Apply3DESP(plr.Character, Color3.fromRGB(255, 100, 100))
                        if tag then tag.Text = plr.Name .. " [" .. math.floor(dist) .. "m]" end
                    else
                        Disable3DESP(plr.Character)
                    end
                end
            end
        end
        if LocalPlayer.Character then Disable3DESP(LocalPlayer.Character) end
    else
        for _, plr in ipairs(Players:GetPlayers()) do
            if plr.Character then Disable3DESP(plr.Character) end
        end
    end

    if Config.BotESPEnabled then
        for i = #TargetCache.Bots, 1, -1 do
            local bot = TargetCache.Bots[i]
            if bot and bot.Parent then
                local mainPart = GetMainPart(bot)
                if mainPart then
                    local dist = (camPos - mainPart.Position).Magnitude
                    if dist <= Config.BotESPDistance then
                        local tag = Apply3DESP(bot, Color3.fromRGB(255, 180, 50))
                        if tag then tag.Text = "[BOT] " .. bot.Name .. " [" .. math.floor(dist) .. "m]" end
                    else
                        Disable3DESP(bot)
                    end
                end
            else
                table.remove(TargetCache.Bots, i)
            end
        end
    else
        for _, bot in ipairs(TargetCache.Bots) do Disable3DESP(bot) end
    end

    local function ProcessListESP(list, isEnabled, color, maxDist, prefix)
        if isEnabled then
            for i = #list, 1, -1 do
                local obj = list[i]
                if obj and obj.Parent then
                    local mainPart = GetMainPart(obj)
                    if mainPart then
                        local dist = (camPos - mainPart.Position).Magnitude
                        if dist <= maxDist then
                            local tag = Apply3DESP(obj, color)
                            if tag then 
                                local realName = GetItemName(obj)
                                tag.Text = prefix .. " " .. realName .. " [" .. math.floor(dist) .. "m]" 
                            end
                        else
                            Disable3DESP(obj)
                        end
                    end
                else
                    table.remove(list, i)
                end
            end
        else
            for _, obj in ipairs(list) do Disable3DESP(obj) end
        end
    end

    ProcessListESP(TargetCache.ItemBoxes, Config.ItemBoxESPEnabled, Color3.fromRGB(50, 255, 200), Config.ItemBoxESPDistance, "[LOOT]")
    ProcessListESP(TargetCache.Exits, Config.ExitESPEnabled, Color3.fromRGB(255, 230, 80), Config.ExitESPDistance, "[SAFEZONE]")
    ProcessListESP(TargetCache.Corpses, Config.CorpseESPEnabled, Color3.fromRGB(180, 180, 200), Config.CorpseESPDistance, "[DEAD]")

    if Config.AimEnabled or Config.BotLockEnabled then
        local targetHead = GetClosestTarget()
        if targetHead then
            local targetPos = targetHead.Position
            if Config.PredictionEnabled and targetHead.AssemblyLinearVelocity then
                targetPos = targetPos + (targetHead.AssemblyLinearVelocity * 0.035)
            end
            
            if Config.Smoothness <= 0.001 then
                Camera.CFrame = CFrame.new(camPos, targetPos)
            else
                Camera.CFrame = Camera.CFrame:Lerp(CFrame.new(camPos, targetPos), Config.Smoothness)
            end
            
            -- ระบบยิงออโต้ (รวมถึงคลิกขวาค้างแล้วยิงออโต้)
            if (Config.AutoShootEnabled or (Config.AutoHoldShootEnabled and IsRightMouseDown)) and mouse1press then
                pcall(function()
                    mouse1press()
                    task.delay(0.01, function() if mouse1release then mouse1release() end end)
                end)
            end
        end
    end
end)

Minimize.MouseButton1Click:Connect(function()
    IsMinimized = not IsMinimized
    ContentContainer.Visible = not IsMinimized
    Footer.Visible = not IsMinimized
    Main.Size = IsMinimized and UDim2.fromOffset(940, 75) or UDim2.fromOffset(940, 720)
    Minimize.Text = IsMinimized and "+" or "-"
end)

Close.MouseButton1Click:Connect(function()
    IsRunning = false
    CleanAllESP()
    ApplyFullBright(false)
    ScreenGui:Destroy()
end)
