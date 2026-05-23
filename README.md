local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local player = Players.LocalPlayer

local DEFAULT_SETTINGS = {
    FLY_UP_HEIGHT = 10,
    FLY_SPEED = 200, 
    WAIT_TIME = 0.28,
    CHECKPOINT_FOLDER = workspace:WaitForChild("Checkpoints", 15)
}

local currentFlySpeed = DEFAULT_SETTINGS.FLY_SPEED
local scanMultiplier = 1 

local running = false
local isMinimized = false
local protectionConnection = nil

local function createLoadingScreen()
    local loadingGui = Instance.new("ScreenGui", player.PlayerGui)
    loadingGui.Name = "Loading_ChienDo"; loadingGui.DisplayOrder = 999
    local bg = Instance.new("Frame", loadingGui); bg.Size = UDim2.new(1, 0, 1, 0); bg.BackgroundColor3 = Color3.new(0, 0, 0); bg.BackgroundTransparency = 0.2; bg.BorderSizePixel = 0
    local label = Instance.new("TextLabel", bg); label.Size = UDim2.new(1, 0, 0, 50); label.Position = UDim2.new(0, 0, 0.5, -25); label.BackgroundTransparency = 1; label.Text = "MADE BY CHIEN DO"; label.TextColor3 = Color3.new(1, 1, 1); label.Font = Enum.Font.GothamBold; label.TextSize = 45; label.ZIndex = 10
    task.spawn(function()
        for i = 1, 120 do
            task.spawn(function()
                local dot = Instance.new("Frame", bg); dot.Size = UDim2.new(0, 5, 0, 5); dot.BackgroundColor3 = Color3.new(1, 0, 0); dot.BorderSizePixel = 0; dot.Position = UDim2.new(math.random(), 0, math.random(), 0); Instance.new("UICorner", dot)
                TweenService:Create(dot, TweenInfo.new(3, Enum.EasingStyle.Linear), {Position = UDim2.new(math.random(), 0, math.random(), 0), BackgroundTransparency = 1}):Play()
            end)
        end
    end)
    task.wait(3); loadingGui:Destroy()
end

local function makeDraggable(gui)
    local dragging, dragInput, dragStart, startPos
    gui.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true; dragStart = input.Position; startPos = gui.Position
        end
    end)
    gui.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then dragInput = input end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if input == dragInput and dragging then
            local delta = input.Position - dragStart
            gui.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end
    end)
    UserInputService.InputEnded:Connect(function(input) if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then dragging = false end end)
end

createLoadingScreen()

local oldGui = player.PlayerGui:FindFirstChild("AutoObby_GodAlways")
if oldGui then oldGui:Destroy() end

local mainGui = Instance.new("ScreenGui", player.PlayerGui)
mainGui.Name = "AutoObby_GodAlways"; mainGui.ResetOnSpawn = false

local frame = Instance.new("Frame", mainGui)
frame.Size = UDim2.new(0, 220, 0, 180); frame.Position = UDim2.new(0, 30, 0.5, -90); frame.BackgroundColor3 = Color3.fromRGB(15, 15, 15); frame.BorderSizePixel = 0; frame.ClipsDescendants = true; frame.Active = true
Instance.new("UICorner", frame); makeDraggable(frame)

task.spawn(function()
    while mainGui.Parent do
        local dot = Instance.new("Frame", frame); dot.Size = UDim2.new(0, 3, 0, 3); dot.BackgroundColor3 = Color3.new(1, 0, 0); dot.Position = UDim2.new(math.random(), 0, math.random(), 0); dot.ZIndex = 1; dot.BorderSizePixel = 0; Instance.new("UICorner", dot)
        TweenService:Create(dot, TweenInfo.new(2, Enum.EasingStyle.Linear), {Position = UDim2.new(math.random(), 0, math.random(), 0), BackgroundTransparency = 1}):Play()
        game:GetService("Debris"):AddItem(dot, 2); task.wait(0.1)
    end
end)

local topContent = Instance.new("Frame", frame); topContent.Size = UDim2.new(1, 0, 0, 70); topContent.BackgroundTransparency = 1; topContent.ZIndex = 5
local title = Instance.new("TextLabel", topContent); title.Size = UDim2.new(1, 0, 0, 30); title.Position = UDim2.new(0, 0, 0, 15); title.Text = "obby for ugc"; title.TextColor3 = Color3.new(1, 1, 1); title.Font = Enum.Font.GothamBold; title.TextSize = 18; title.BackgroundTransparency = 1; title.ZIndex = 6
local subTitle = Instance.new("TextLabel", topContent); subTitle.Size = UDim2.new(1, 0, 0, 20); subTitle.Position = UDim2.new(0, 0, 0, 40); subTitle.Text = "(AFK OR PLAY)"; subTitle.TextColor3 = Color3.fromRGB(200, 200, 200); subTitle.Font = Enum.Font.Gotham; subTitle.TextSize = 14; subTitle.BackgroundTransparency = 1; subTitle.ZIndex = 6

local bottomContent = Instance.new("Frame", frame); bottomContent.Size = UDim2.new(1, 0, 0, 110); bottomContent.Position = UDim2.new(0, 0, 0, 70); bottomContent.BackgroundTransparency = 1; bottomContent.ZIndex = 5
local btn = Instance.new("TextButton", bottomContent); btn.Size = UDim2.new(0, 180, 0, 50); btn.Position = UDim2.new(0.5, -90, 0, 20); btn.Text = "Auto Farm Stage: OFF"; btn.BackgroundColor3 = Color3.new(1, 1, 1); btn.TextColor3 = Color3.new(0, 0, 0); btn.Font = Enum.Font.GothamBold; btn.TextSize = 15; btn.ZIndex = 6; btn.Active = true
Instance.new("UICorner", btn)

local toggleBtn = Instance.new("TextButton", frame); toggleBtn.Size = UDim2.new(0, 25, 0, 25); toggleBtn.Position = UDim2.new(1, -30, 0, 5); toggleBtn.Text = "▲"; toggleBtn.TextColor3 = Color3.new(1, 1, 1); toggleBtn.BackgroundTransparency = 1; toggleBtn.Font = Enum.Font.GothamBold; toggleBtn.TextSize = 18; toggleBtn.ZIndex = 10

toggleBtn.MouseButton1Click:Connect(function()
    isMinimized = not isMinimized
    if isMinimized then
        frame:TweenSize(UDim2.new(0, 220, 0, 70), "Out", "Quart", 0.3, true); toggleBtn.Text = "▼"; bottomContent.Visible = false
    else
        frame:TweenSize(UDim2.new(0, 220, 0, 180), "Out", "Quart", 0.3, true); toggleBtn.Text = "▲"; bottomContent.Visible = true
    end
end)

local function toggleProtection(state)
    if state then
        if not protectionConnection then
            protectionConnection = RunService.Stepped:Connect(function()
                if player.Character then
                    local h = player.Character:FindFirstChildOfClass("Humanoid")
                    if h then h.MaxHealth = math.huge; h.Health = math.huge; h:SetStateEnabled(Enum.HumanoidStateType.Dead, false) end
                    for _, p in pairs(player.Character:GetDescendants()) do if p:IsA("BasePart") then p.CanCollide = false end end
                end
            end)
        end
    else
        if protectionConnection then protectionConnection:Disconnect(); protectionConnection = nil end
    end
end

local function flyToTarget(target)
    local char = player.Character; local hrp = char and char:FindFirstChild("HumanoidRootPart")
    if not hrp or not target then return end
    
    local touched = false
    local conn; conn = target.Touched:Connect(function(hit) 
        if hit:IsDescendantOf(char) then 
            touched = true 
            if conn then conn:Disconnect(); conn = nil end 
        end 
    end)
    
    local targetPos = target.Position 

    local upY = hrp.Position.Y + DEFAULT_SETTINGS.FLY_UP_HEIGHT
    while running and not touched and hrp.Position.Y < upY - 1 do 
        hrp.Velocity = Vector3.new(0, 50, 0); task.wait() 
    end
    
    while running and not touched and (Vector2.new(hrp.Position.X, hrp.Position.Z) - Vector2.new(targetPos.X, targetPos.Z)).Magnitude > 4 do
        hrp.Velocity = (Vector3.new(targetPos.X, hrp.Position.Y, targetPos.Z) - hrp.Position).Unit * currentFlySpeed
        task.wait()
    end
    
    local t = tick()
    while running and not touched do
        hrp.Velocity = (targetPos - hrp.Position).Unit * (currentFlySpeed * 0.4)
        task.wait()
        if tick() - t > 6 then 
            hrp.CFrame = CFrame.new(targetPos); 
            touched = true 
        end
    end

    hrp.Velocity = Vector3.new(0, 0.2, 0)
    if conn then conn:Disconnect(); conn = nil end 
    
    currentFlySpeed = DEFAULT_SETTINGS.FLY_SPEED
    scanMultiplier = 1
    
    task.wait(DEFAULT_SETTINGS.WAIT_TIME) 
end

btn.MouseButton1Click:Connect(function()
    if running then
        running = false; toggleProtection(false)
        btn.Text = "Auto Farm Stage: OFF"; btn.BackgroundColor3 = Color3.new(1, 1, 1); btn.TextColor3 = Color3.new(0, 0, 0)
    else
        running = true; toggleProtection(true)
        btn.Text = "Auto Farm Stage: ON"; btn.BackgroundColor3 = Color3.new(1, 0, 0); btn.TextColor3 = Color3.new(1, 1, 1)
        
        task.spawn(function()
            local lastFoundTime = tick()
            while running do
                local hrp = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
                if not hrp then task.wait(1) continue end
                
                local closestStageNum = -1
                local minDist = math.huge
                
                for _, cp in pairs(DEFAULT_SETTINGS.CHECKPOINT_FOLDER:GetChildren()) do
                    local stageNum = tonumber(cp.Name)
                    if stageNum and cp:IsA("BasePart") then
                        local d = (hrp.Position - cp.Position).Magnitude
                        if d < minDist then
                            minDist = d
                            closestStageNum = stageNum
                        end
                    end
                end
                
                local nextTarget = DEFAULT_SETTINGS.CHECKPOINT_FOLDER:FindFirstChild(tostring(closestStageNum + 1))
                
                if nextTarget then
                    lastFoundTime = tick()
                    flyToTarget(nextTarget)
                else
                    if tick() - lastFoundTime > 5 then
                        scanMultiplier = 2
                        currentFlySpeed = DEFAULT_SETTINGS.FLY_SPEED * 1.5
                        lastFoundTime = tick()
                    end
                    task.wait(0.5)
                end
            end
        end)
    end
end)

player.CharacterAdded:Connect(function() 
    running = false; toggleProtection(false)
    currentFlySpeed = DEFAULT_SETTINGS.FLY_SPEED
    scanMultiplier = 1
    btn.Text = "Auto Farm Stage: OFF"; btn.BackgroundColor3 = Color3.new(1, 1, 1) 
end)
