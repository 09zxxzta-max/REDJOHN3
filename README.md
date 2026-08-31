local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local CoreGui = game:GetService("CoreGui")
local TweenService = game:GetService("TweenService")
local VirtualUser = game:GetService("VirtualUser")

local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera

local ContainerParent = gethui and gethui() or (CoreGui:FindFirstChild("RobloxGui") and CoreGui or LocalPlayer:WaitForChild("PlayerGui"))
if ContainerParent:FindFirstChild("TargetLockHub_Protected") then
    ContainerParent:FindFirstChild("TargetLockHub_Protected"):Destroy()
end

local BG_Main = Color3.fromRGB(15, 15, 18)
local BG_Header = Color3.fromRGB(22, 22, 26)
local Card_BG = Color3.fromRGB(28, 28, 34)
local Card_Active = Color3.fromRGB(55, 55, 65)
local Stroke_Color = Color3.fromRGB(45, 45, 55)

local Text_Main = Color3.fromRGB(240, 240, 245)
local Text_Sub = Color3.fromRGB(140, 140, 150)
local Accent_Status = Color3.fromRGB(200, 200, 210)

local AimEnabled = true
local ESPEnabled = true
local WallCheckEnabled = true
local SnapLockMode = true
local PredictionEnabled = true
local AutoShootEnabled = false
local SpeedEnabled = false
local SpeedValue = 50

local ShowFOVCircle = true
local FOV = 180
local Smoothness = 0.25

local TargetModes = { "Head", "Body", "Hybrid" }
local CurrentTargetModeIndex = 3
local TargetMode = TargetModes[CurrentTargetModeIndex]

local BulletSpeed = 1000
local CurrentTarget = nil
local CurrentTargetPart = nil
local IsMinimized = false
local LastShotTime = 0

pcall(function()
    local rawMetatable = getrawmetatable(game)
    local oldIndex = rawMetatable.__index
    setreadonly(rawMetatable, false)

    rawMetatable.__index = newcclosure(function(self, key)
        if not checkcaller() and self:IsA("Humanoid") and key == "WalkSpeed" then
            return 16
        end
        return oldIndex(self, key)
    end)

    setreadonly(rawMetatable, true)
end)

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "TargetLockHub_Protected"
ScreenGui.ResetOnSpawn = false
ScreenGui.Parent = ContainerParent

local ESPFolder = Instance.new("Folder")
ESPFolder.Name = "ESPHolder"
ESPFolder.Parent = ScreenGui

local Main = Instance.new("Frame")
Main.Size = UDim2.fromOffset(290, 560)
Main.Position = UDim2.new(0.5, -145, 0.5, -280)
Main.BackgroundColor3 = BG_Main
Main.BorderSizePixel = 0
Main.Active = true
Main.ClipsDescendants = true
Main.Parent = ScreenGui

local Corner = Instance.new("UICorner")
Corner.CornerRadius = UDim.new(0, 10)
Corner.Parent = Main

local MainStroke = Instance.new("UIStroke")
MainStroke.Thickness = 1
MainStroke.Color = Stroke_Color
MainStroke.Transparency = 0.2
MainStroke.Parent = Main

local dragging, dragInput, dragStart, startPos
local function updateDrag(input)
    local delta = input.Position - dragStart
    Main.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
end

Main.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        dragging = true
        dragStart = input.Position
        startPos = Main.Position
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then dragging = false end
        end)
    end
end)

Main.InputChanged:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
        dragInput = input
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if input == dragInput and dragging then updateDrag(input) end
end)

local Header = Instance.new("Frame")
Header.Size = UDim2.new(1, 0, 0, 42)
Header.BackgroundColor3 = BG_Header
Header.BorderSizePixel = 0
Header.Parent = Main

local HeaderCorner = Instance.new("UICorner")
HeaderCorner.CornerRadius = UDim.new(0, 10)
HeaderCorner.Parent = Header

local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, -80, 1, 0)
Title.Position = UDim2.fromOffset(15, 0)
Title.BackgroundTransparency = 1
Title.Text = "REDJOHN"
Title.TextColor3 = Text_Main
Title.TextSize = 13
Title.Font = Enum.Font.GothamBold
Title.TextXAlignment = Enum.TextXAlignment.Left
Title.Parent = Header

local Minimize = Instance.new("TextButton")
Minimize.Size = UDim2.fromOffset(26, 26)
Minimize.Position = UDim2.new(1, -60, 0.5, -13)
Minimize.BackgroundColor3 = Card_BG
Minimize.Text = "-"
Minimize.TextColor3 = Text_Main
Minimize.TextSize = 16
Minimize.Font = Enum.Font.GothamBold
Minimize.Parent = Header
Instance.new("UICorner", Minimize).CornerRadius = UDim.new(0, 6)

local Close = Instance.new("TextButton")
Close.Size = UDim2.fromOffset(26, 26)
Close.Position = UDim2.new(1, -30, 0.5, -13)
Close.BackgroundColor3 = Color3.fromRGB(40, 20, 20)
Close.Text = "X"
Close.TextColor3 = Color3.fromRGB(220, 90, 90)
Close.TextSize = 13
Close.Font = Enum.Font.GothamBold
Close.Parent = Header
Instance.new("UICorner", Close).CornerRadius = UDim.new(0, 6)

local Content = Instance.new("Frame")
Content.Size = UDim2.new(1, -24, 1, -54)
Content.Position = UDim2.fromOffset(12, 48)
Content.BackgroundTransparency = 1
Content.Parent = Main

local Layout = Instance.new("UIListLayout")
Layout.SortOrder = Enum.SortOrder.LayoutOrder
Layout.Padding = UDim.new(0, 5)
Layout.Parent = Content

local function CreateButton(text, active, order, parentFrame)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(1, 0, 0, 30)
    btn.BackgroundColor3 = active and Card_Active or Card_BG
    btn.Text = text
    btn.TextColor3 = active and Text_Main or Text_Sub
    btn.TextSize = 11
    btn.Font = Enum.Font.GothamBold
    btn.LayoutOrder = order
    btn.Parent = parentFrame or Content
    
    local stroke = Instance.new("UIStroke")
    stroke.Thickness = 1
    stroke.Color = Stroke_Color
    stroke.Transparency = 0.5
    stroke.Parent = btn

    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 6)
    return btn
end

local function CreateSlider(title, minVal, maxVal, startVal, isFloat, order, callback)
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1, 0, 0, 42)
    frame.BackgroundColor3 = Card_BG
    frame.BorderSizePixel = 0
    frame.LayoutOrder = order
    frame.Parent = Content
    Instance.new("UICorner", frame).CornerRadius = UDim.new(0, 6)

    local stroke = Instance.new("UIStroke")
    stroke.Thickness = 1
    stroke.Color = Stroke_Color
    stroke.Transparency = 0.5
    stroke.Parent = frame

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1, -15, 0, 20)
    label.Position = UDim2.fromOffset(10, 2)
    label.BackgroundTransparency = 1
    label.Text = title .. ": " .. tostring(startVal)
    label.TextColor3 = Text_Main
    label.TextSize = 11
    label.Font = Enum.Font.GothamBold
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.Parent = frame

    local sliderBar = Instance.new("Frame")
    sliderBar.Size = UDim2.new(1, -20, 0, 6)
    sliderBar.Position = UDim2.new(0, 10, 0, 26)
    sliderBar.BackgroundColor3 = BG_Header
    sliderBar.BorderSizePixel = 0
    sliderBar.Parent = frame
    Instance.new("UICorner", sliderBar).CornerRadius = UDim.new(1, 0)

    local sliderFill = Instance.new("Frame")
    sliderFill.Size = UDim2.new((startVal - minVal) / (maxVal - minVal), 0, 1, 0)
    sliderFill.BackgroundColor3 = Card_Active
    sliderFill.BorderSizePixel = 0
    sliderFill.Parent = sliderBar
    Instance.new("UICorner", sliderFill).CornerRadius = UDim.new(1, 0)

    local draggingSlider = false
    local function updateValue(input)
        local pos = math.clamp((input.Position.X - sliderBar.AbsolutePosition.X) / sliderBar.AbsoluteSize.X, 0, 1)
        sliderFill.Size = UDim2.new(pos, 0, 1, 0)
        
        local val
        if isFloat then
            val = tonumber(string.format("%.2f", minVal + ((maxVal - minVal) * pos)))
            label.Text = title .. ": " .. string.format("%.2f", val)
        else
            val = math.floor(minVal + ((maxVal - minVal) * pos))
            label.Text = title .. ": " .. val
        end
        callback(val)
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

    return frame
end

local AimBtn = CreateButton("Aim Lock: ON", true, 1)
local ESPBtn = CreateButton("ESP Vision: ON", true, 2)
local SnapBtn = CreateButton("Snap Mode: ON", true, 3)
local PredictBtn = CreateButton("Prediction: ON", true, 4)
local AutoShootBtn = CreateButton("Auto Shoot: OFF", false, 5)
local TargetModeBtn = CreateButton("Target Part: Hybrid", true, 6)
local FOVToggleBtn = CreateButton("FOV Circle: ON", true, 7)

CreateSlider("FOV Size", 30, 400, FOV, false, 8, function(v)
    FOV = v
end)

CreateSlider("Smooth Aim", 0.01, 1.0, Smoothness, true, 9, function(v)
    Smoothness = v
end)

CreateSlider("Speed Value", 20, 200, SpeedValue, false, 10, function(v)
    SpeedValue = v
end)

local SpeedToggleBtn = CreateButton("Speed System: OFF", false, 11)

local StatusLabel = Instance.new("TextLabel")
StatusLabel.Size = UDim2.new(1, 0, 0, 30)
StatusLabel.BackgroundColor3 = BG_Header
StatusLabel.Text = "Status: Safe Idle"
StatusLabel.TextColor3 = Text_Sub
StatusLabel.TextSize = 11
StatusLabel.Font = Enum.Font.Gotham
StatusLabel.LayoutOrder = 12
StatusLabel.Parent = Content
Instance.new("UICorner", StatusLabel).CornerRadius = UDim.new(0, 6)

local StatusStroke = Instance.new("UIStroke")
StatusStroke.Thickness = 1
StatusStroke.Color = Stroke_Color
StatusStroke.Parent = StatusLabel

local FOVCircle = Instance.new("Frame")
FOVCircle.Size = UDim2.fromOffset(FOV * 2, FOV * 2)
FOVCircle.AnchorPoint = Vector2.new(0.5, 0.5)
FOVCircle.Position = UDim2.fromScale(0.5, 0.5)
FOVCircle.BackgroundTransparency = 1
FOVCircle.Parent = ScreenGui

local FOVStroke = Instance.new("UIStroke")
FOVStroke.Thickness = 1
FOVStroke.Color = Color3.fromRGB(150, 150, 160)
FOVStroke.Transparency = 0.6
FOVStroke.Parent = FOVCircle
Instance.new("UICorner", FOVCircle).CornerRadius = UDim.new(1, 0)

local function RemoveESP(player)
    local hl = ESPFolder:FindFirstChild("HL_" .. player.Name)
    if hl then hl:Destroy() end
    local bb = ESPFolder:FindFirstChild("BB_" .. player.Name)
    if bb then bb:Destroy() end
end

local function UpdateESP()
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character then
            local char = player.Character
            local head = char:FindFirstChild("Head")
            local hum = char:FindFirstChildOfClass("Humanoid")
            local root = char:FindFirstChild("HumanoidRootPart")

            if ESPEnabled and head and hum and root and hum.Health > 0 then
                local hl = ESPFolder:FindFirstChild("HL_" .. player.Name) or Instance.new("Highlight")
                hl.Name = "HL_" .. player.Name
                hl.Adornee = char
                hl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
                hl.FillColor = (player == CurrentTarget) and Color3.fromRGB(200, 200, 210) or Color3.fromRGB(80, 80, 95)
                hl.FillTransparency = 0.6
                hl.OutlineColor = Color3.fromRGB(255, 255, 255)
                hl.OutlineTransparency = 0.5
                hl.Enabled = true
                hl.Parent = ESPFolder

                local bb = ESPFolder:FindFirstChild("BB_" .. player.Name) or Instance.new("BillboardGui")
                bb.Name = "BB_" .. player.Name
                bb.Adornee = head
                bb.Size = UDim2.fromOffset(120, 30)
                bb.StudsOffset = Vector3.new(0, 2.2, 0)
                bb.AlwaysOnTop = true
                bb.Enabled = true
                bb.Parent = ESPFolder

                local txt = bb:FindFirstChild("T") or Instance.new("TextLabel")
                txt.Name = "T"
                txt.Size = UDim2.new(1, 0, 1, 0)
                txt.BackgroundTransparency = 1
                txt.TextColor3 = Text_Main
                txt.TextStrokeTransparency = 0.4
                txt.TextSize = 11
                txt.Font = Enum.Font.GothamBold

                local myRoot = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
                local dist = myRoot and math.floor((myRoot.Position - root.Position).Magnitude) or 0
                txt.Text = player.DisplayName .. " [" .. dist .. "m]"
                txt.Parent = bb
            else
                RemoveESP(player)
            end
        end
    end
end

local function IsVisible(targetPart)
    if not WallCheckEnabled then return true end
    local rayParams = RaycastParams.new()
    rayParams.FilterType = Enum.RaycastFilterType.Exclude
    rayParams.FilterDescendantsInstances = { LocalPlayer.Character, targetPart.Parent }

    local hit = workspace:Raycast(Camera.CFrame.Position, targetPart.Position - Camera.CFrame.Position, rayParams)
    return hit == nil
end

local function GetTargetPart(char)
    if not char then return nil end
    local head = char:FindFirstChild("Head")
    local torso = char:FindFirstChild("UpperTorso") or char:FindFirstChild("Torso")

    if TargetMode == "Head" then return head or torso end
    if TargetMode == "Body" then return torso or head end
    return (math.random() <= 0.35) and (head or torso) or (torso or head)
end

local function GetTargetPosition(part)
    if not part then return nil end
    local pos = part.Position
    if PredictionEnabled then
        local root = part.Parent:FindFirstChild("HumanoidRootPart")
        if root then
            local vel = root.AssemblyLinearVelocity or root.Velocity
            local dist = (Camera.CFrame.Position - pos).Magnitude
            pos = pos + (vel * (dist / BulletSpeed))
        end
    end

    local jitter = Vector3.new((math.random() - 0.5) * 0.1, (math.random() - 0.5) * 0.1, (math.random() - 0.5) * 0.1)
    return pos + jitter
end

local function GetClosestPlayer()
    local center = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
    local closestPlayer, closestPart = nil, nil
    local minDistance = FOV

    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character then
            local char = player.Character
            local hum = char:FindFirstChildOfClass("Humanoid")
            if hum and hum.Health > 0 then
                local part = GetTargetPart(char)
                if part and IsVisible(part) then
                    local targetPos = GetTargetPosition(part)
                    local screenPos, onScreen = Camera:WorldToViewportPoint(targetPos)
                    if onScreen then
                        local dist = (Vector2.new(screenPos.X, screenPos.Y) - center).Magnitude
                        if dist < minDistance then
                            minDistance = dist
                            closestPlayer = player
                            closestPart = part
                        end
                    end
                end
            end
        end
    end
    return closestPlayer, closestPart
end

RunService.Heartbeat:Connect(function()
    UpdateESP()

    if SpeedEnabled and LocalPlayer.Character then
        local hum = LocalPlayer.Character:FindFirstChildOfClass("Humanoid")
        local root = LocalPlayer.Character:FindFirstChild("HumanoidRootPart")

        if hum and root and hum.MoveDirection.Magnitude > 0 then
            local speedJitter = SpeedValue + (math.random(-15, 15) * 0.1)
            local currentVel = root.AssemblyLinearVelocity
            local newVel = hum.MoveDirection * speedJitter
            root.AssemblyLinearVelocity = Vector3.new(newVel.X, currentVel.Y, newVel.Z)
        end
    end

    FOVCircle.Size = UDim2.fromOffset(FOV * 2, FOV * 2)
    FOVCircle.Visible = AimEnabled and ShowFOVCircle

    if not AimEnabled then
        StatusLabel.Text = "Status: Disabled"
        StatusLabel.TextColor3 = Text_Sub
        CurrentTarget = nil
        return
    end

    CurrentTarget, CurrentTargetPart = GetClosestPlayer()

    if CurrentTarget and CurrentTargetPart then
        StatusLabel.Text = "Locked: " .. CurrentTarget.DisplayName
        StatusLabel.TextColor3 = Accent_Status

        local targetPos = GetTargetPosition(CurrentTargetPart)
        local targetCFrame = CFrame.lookAt(Camera.CFrame.Position, targetPos)

        local dynamicSmoothness = SnapLockMode and (Smoothness + 0.1) or Smoothness
        Camera.CFrame = Camera.CFrame:Lerp(targetCFrame, math.clamp(dynamicSmoothness, 0.01, 1))

        if AutoShootEnabled then
            local currentTime = tick()
            local randomDelay = 0.08 + (math.random() * 0.06)
            if currentTime - LastShotTime >= randomDelay then
                LastShotTime = currentTime
                VirtualUser:Button1Down(Vector2.new(0, 0), Camera.CFrame)
                task.delay(0.03, function()
                    VirtualUser:Button1Up(Vector2.new(0, 0), Camera.CFrame)
                end)
            end
        end
    else
        StatusLabel.Text = "Status: Searching Target..."
        StatusLabel.TextColor3 = Text_Sub
    end
end)

local function ToggleState(btn, state, label)
    btn.BackgroundColor3 = state and Card_Active or Card_BG
    btn.TextColor3 = state and Text_Main or Text_Sub
    btn.Text = label .. ": " .. (state and "ON" or "OFF")
end

AimBtn.MouseButton1Click:Connect(function()
    AimEnabled = not AimEnabled
    ToggleState(AimBtn, AimEnabled, "Aim Lock")
end)

ESPBtn.MouseButton1Click:Connect(function()
    ESPEnabled = not ESPEnabled
    ToggleState(ESPBtn, ESPEnabled, "ESP Vision")
    if not ESPEnabled then
        for _, p in ipairs(Players:GetPlayers()) do RemoveESP(p) end
    end
end)

SnapBtn.MouseButton1Click:Connect(function()
    SnapLockMode = not SnapLockMode
    ToggleState(SnapBtn, SnapLockMode, "Snap Mode")
end)

PredictBtn.MouseButton1Click:Connect(function()
    PredictionEnabled = not PredictionEnabled
    ToggleState(PredictBtn, PredictionEnabled, "Prediction")
end)

AutoShootBtn.MouseButton1Click:Connect(function()
    AutoShootEnabled = not AutoShootEnabled
    ToggleState(AutoShootBtn, AutoShootEnabled, "Auto Shoot")
end)

TargetModeBtn.MouseButton1Click:Connect(function()
    CurrentTargetModeIndex = (CurrentTargetModeIndex % #TargetModes) + 1
    TargetMode = TargetModes[CurrentTargetModeIndex]
    TargetModeBtn.Text = "Target Part: " .. TargetMode
end)

FOVToggleBtn.MouseButton1Click:Connect(function()
    ShowFOVCircle = not ShowFOVCircle
    ToggleState(FOVToggleBtn, ShowFOVCircle, "FOV Circle")
end)

SpeedToggleBtn.MouseButton1Click:Connect(function()
    SpeedEnabled = not SpeedEnabled
    ToggleState(SpeedToggleBtn, SpeedEnabled, "Speed System")
end)

Minimize.MouseButton1Click:Connect(function()
    IsMinimized = not IsMinimized
    Content.Visible = not IsMinimized
    
    TweenService:Create(Main, TweenInfo.new(0.25, Enum.EasingStyle.Quart, Enum.EasingDirection.Out), {
        Size = IsMinimized and UDim2.fromOffset(290, 42) or UDim2.fromOffset(290, 560)
    }):Play()
end)

Close.MouseButton1Click:Connect(function()
    for _, p in ipairs(Players:GetPlayers()) do RemoveESP(p) end
    ScreenGui:Destroy()
end)
