# REDJOHN3
--// Target Lock Hub - Dark Charcoal Edition (Anti-Ban Protected)
--// Theme: Slate Dark / Matte Grey with Speed & Protection System

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local CoreGui = game:GetService("CoreGui")
local TweenService = game:GetService("TweenService")
local VirtualUser = game:GetService("VirtualUser")

local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera

-- Safe Parent Detection
local ContainerParent = gethui and gethui() or (CoreGui:FindFirstChild("RobloxGui") and CoreGui or LocalPlayer:WaitForChild("PlayerGui"))
if ContainerParent:FindFirstChild("TargetLockHub_Protected") then
    ContainerParent:FindFirstChild("TargetLockHub_Protected"):Destroy()
end

--==================================================
-- DARK & GREY PALETTE
--==================================================
local BG_Main = Color3.fromRGB(15, 15, 18)
local BG_Header = Color3.fromRGB(22, 22, 26)
local Card_BG = Color3.fromRGB(28, 28, 34)
local Card_Active = Color3.fromRGB(55, 55, 65)
local Stroke_Color = Color3.fromRGB(45, 45, 55)

local Text_Main = Color3.fromRGB(240, 240, 245)
local Text_Sub = Color3.fromRGB(140, 140, 150)
local Accent_Status = Color3.fromRGB(200, 200, 210)

--==================================================
-- SETTINGS & STATE
--==================================================
local AimEnabled = true
local ESPEnabled = true
local WallCheckEnabled = true
local SnapLockMode = true
local PredictionEnabled = true
local AutoShootEnabled = false

-- Speed System Settings
local SpeedEnabled = false
local SpeedValue = 50

local TargetModes = { "Head", "Body", "Hybrid" }
local CurrentTargetModeIndex = 3
local TargetMode = TargetModes[CurrentTargetModeIndex]

local FOV = 180
local BaseSmoothness = 0.25
local BulletSpeed = 1000

local CurrentTarget = nil
local CurrentTargetPart = nil
local IsMinimized = false
local LastShotTime = 0

--==================================================
-- ANTI-BAN METATABLE HOOK (Client Protection)
--==================================================
pcall(function()
    local rawMetatable = getrawmetatable(game)
    local oldIndex = rawMetatable.__index
    local oldNamecall = rawMetatable.__namecall
    setreadonly(rawMetatable, false)

    -- ซ่อนค่า WalkSpeed ที่ผิดปกติจากการถูกสแกน
    rawMetatable.__index = newcclosure(function(self, key)
        if not checkcaller() and self:IsA("Humanoid") and key == "WalkSpeed" then
            return 16
        end
        return oldIndex(self, key)
    end)

    setreadonly(rawMetatable, true)
end)

--==================================================
-- GUI CREATION
--==================================================
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "TargetLockHub_Protected"
ScreenGui.ResetOnSpawn = false
ScreenGui.Parent = ContainerParent

local ESPFolder = Instance.new("Folder")
ESPFolder.Name = "ESPHolder"
ESPFolder.Parent = ScreenGui

-- Main Container
local Main = Instance.new("Frame")
Main.Size = UDim2.fromOffset(290, 500)
Main.Position = UDim2.new(0.5, -145, 0.5, -250)
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

-- Drag Engine
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

-- Top Header
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
Title.Text = "TARGET LOCK (SAFE)"
Title.TextColor3 = Text_Main
Title.TextSize = 13
Title.Font = Enum.Font.GothamBold
Title.TextXAlignment = Enum.TextXAlignment.Left
Title.Parent = Header

-- ปุ่มย่อส่วน (-)
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

-- ปุ่มปิด (X)
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

-- Content Area
local Content = Instance.new("Frame")
Content.Size = UDim2.new(1, -24, 1, -54)
Content.Position = UDim2.fromOffset(12, 48)
Content.BackgroundTransparency = 1
Content.Parent = Main

local Layout = Instance.new("UIListLayout")
Layout.SortOrder = Enum.SortOrder.LayoutOrder
Layout.Padding = UDim.new(0, 7)
Layout.Parent = Content

-- Helper: Create Dark Button
local function CreateButton(text, active, order, parentFrame)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(1, 0, 0, 34)
    btn.BackgroundColor3 = active and Card_Active or Card_BG
    btn.Text = text
    btn.TextColor3 = active and Text_Main or Text_Sub
    btn.TextSize = 12
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

local AimBtn = CreateButton("Aim Lock: ON", true, 1)
local ESPBtn = CreateButton("ESP Vision: ON", true, 2)
local SnapBtn = CreateButton("Snap Mode: ON", true, 3)
local PredictBtn = CreateButton("Prediction: ON", true, 4)
local AutoShootBtn = CreateButton("Auto Shoot: OFF", false, 5)
local TargetModeBtn = CreateButton("Target Part: Hybrid", true, 6)

-- Speed Controls UI Block
local SpeedToggleBtn = CreateButton("Speed System: OFF", false, 7)

local SpeedAdjustFrame = Instance.new("Frame")
SpeedAdjustFrame.Size = UDim2.new(1, 0, 0, 34)
SpeedAdjustFrame.BackgroundTransparency = 1
SpeedAdjustFrame.LayoutOrder = 8
SpeedAdjustFrame.Parent = Content

local SpeedMinusBtn = Instance.new("TextButton")
SpeedMinusBtn.Size = UDim2.new(0.2, 0, 1, 0)
SpeedMinusBtn.BackgroundColor3 = Card_BG
SpeedMinusBtn.Text = "-"
SpeedMinusBtn.TextColor3 = Text_Main
SpeedMinusBtn.TextSize = 14
SpeedMinusBtn.Font = Enum.Font.GothamBold
SpeedMinusBtn.Parent = SpeedAdjustFrame
Instance.new("UICorner", SpeedMinusBtn).CornerRadius = UDim.new(0, 6)

local SpeedDisplayLabel = Instance.new("TextLabel")
SpeedDisplayLabel.Size = UDim2.new(0.56, 0, 1, 0)
SpeedDisplayLabel.Position = UDim2.new(0.22, 0, 0, 0)
SpeedDisplayLabel.BackgroundColor3 = BG_Header
SpeedDisplayLabel.Text = "Speed: " .. SpeedValue
SpeedDisplayLabel.TextColor3 = Text_Main
SpeedDisplayLabel.TextSize = 11
SpeedDisplayLabel.Font = Enum.Font.GothamBold
SpeedDisplayLabel.Parent = SpeedAdjustFrame
Instance.new("UICorner", SpeedDisplayLabel).CornerRadius = UDim.new(0, 6)

local SpeedPlusBtn = Instance.new("TextButton")
SpeedPlusBtn.Size = UDim2.new(0.2, 0, 1, 0)
SpeedPlusBtn.Position = UDim2.new(0.8, 0, 0, 0)
SpeedPlusBtn.BackgroundColor3 = Card_BG
SpeedPlusBtn.Text = "+"
SpeedPlusBtn.TextColor3 = Text_Main
SpeedPlusBtn.TextSize = 14
SpeedPlusBtn.Font = Enum.Font.GothamBold
SpeedPlusBtn.Parent = SpeedAdjustFrame
Instance.new("UICorner", SpeedPlusBtn).CornerRadius = UDim.new(0, 6)

-- Status Bar
local StatusLabel = Instance.new("TextLabel")
StatusLabel.Size = UDim2.new(1, 0, 0, 32)
StatusLabel.BackgroundColor3 = BG_Header
StatusLabel.Text = "Status: Safe Idle"
StatusLabel.TextColor3 = Text_Sub
StatusLabel.TextSize = 11
StatusLabel.Font = Enum.Font.Gotham
StatusLabel.LayoutOrder = 9
StatusLabel.Parent = Content
Instance.new("UICorner", StatusLabel).CornerRadius = UDim.new(0, 6)

local StatusStroke = Instance.new("UIStroke")
StatusStroke.Thickness = 1
StatusStroke.Color = Stroke_Color
StatusStroke.Parent = StatusLabel

-- Minimal Dark FOV Circle
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

--==================================================
-- ESP SYSTEM
--==================================================
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

--==================================================
-- AIM LOGIC (Safe & Humanized)
--==================================================
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
    -- Humanized Random Offset (ป้องกันการเล็งพิกัดตรงเป๊ะ 100% ตลอดเวลา)
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

--==================================================
-- MAIN LOOP (Heartbeat with Anti-Ban Logic)
--==================================================
RunService.Heartbeat:Connect(function()
    UpdateESP()

    -- Safe Speed System (Anti-Velocity Freeze detection)
    if SpeedEnabled and LocalPlayer.Character then
        local hum = LocalPlayer.Character:FindFirstChildOfClass("Humanoid")
        local root = LocalPlayer.Character:FindFirstChild("HumanoidRootPart")

        if hum and root and hum.MoveDirection.Magnitude > 0 then
            -- สุ่ม Jitter ความเร็วเล็กน้อยเพื่อขัดขวางการตรวจจับค่าคงที่
            local speedJitter = SpeedValue + (math.random(-15, 15) * 0.1)
            local currentVel = root.AssemblyLinearVelocity
            local newVel = hum.MoveDirection * speedJitter
            root.AssemblyLinearVelocity = Vector3.new(newVel.X, currentVel.Y, newVel.Z)
        end
    end

    FOVCircle.Size = UDim2.fromOffset(FOV * 2, FOV * 2)
    FOVCircle.Visible = AimEnabled

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

        -- Humanized Aim Lerping (สุ่มค่าความสมูทเพื่อความเนียน)
        local safeSmoothness = SnapLockMode and (0.35 + (math.random() * 0.05)) or (BaseSmoothness + (math.random() * 0.03))
        Camera.CFrame = Camera.CFrame:Lerp(targetCFrame, safeSmoothness)

        -- Humanized Auto Shoot (ป้องกัน Auto Clicker Ban)
        if AutoShootEnabled then
            local currentTime = tick()
            local randomDelay = 0.08 + (math.random() * 0.06) -- สุ่มหน่วงเวลารอบการยิง
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

--==================================================
-- BUTTON EVENTS
--==================================================
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

SpeedToggleBtn.MouseButton1Click:Connect(function()
    SpeedEnabled = not SpeedEnabled
    ToggleState(SpeedToggleBtn, SpeedEnabled, "Speed System")
end)

SpeedMinusBtn.MouseButton1Click:Connect(function()
    if SpeedValue > 20 then
        SpeedValue = SpeedValue - 5
        SpeedDisplayLabel.Text = "Speed: " .. SpeedValue
    end
end)

SpeedPlusBtn.MouseButton1Click:Connect(function()
    if SpeedValue < 200 then
        SpeedValue = SpeedValue + 5
        SpeedDisplayLabel.Text = "Speed: " .. SpeedValue
    end
end)

Minimize.MouseButton1Click:Connect(function()
    IsMinimized = not IsMinimized
    Content.Visible = not IsMinimized
    
    TweenService:Create(Main, TweenInfo.new(0.25, Enum.EasingStyle.Quart, Enum.EasingDirection.Out), {
        Size = IsMinimized and UDim2.fromOffset(290, 42) or UDim2.fromOffset(290, 500)
    }):Play()
end)

Close.MouseButton1Click:Connect(function()
    for _, p in ipairs(Players:GetPlayers()) do RemoveESP(p) end
    ScreenGui:Destroy()
end)
