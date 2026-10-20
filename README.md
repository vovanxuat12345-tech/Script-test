local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local TeleportService = game:GetService("TeleportService")
local GuiService = game:GetService("GuiService")
local StarterGui = game:GetService("StarterGui")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local VirtualUser = game:GetService("VirtualUser")
local player = Players.LocalPlayer
local camera = workspace.CurrentCamera

if _G.ScriptRunning then return end
_G.ScriptRunning = true 

local oldGui = player:WaitForChild("PlayerGui"):FindFirstChild("AutoObby_GodAlways")
if oldGui then oldGui:Destroy() end
local oldLoading = player:WaitForChild("PlayerGui"):FindFirstChild("Loading_ChienDo")
if oldLoading then oldLoading:Destroy() end
local oldAFK = player:WaitForChild("PlayerGui"):FindFirstChild("AFK_FullScreen_Saver")
if oldAFK then oldAFK:Destroy() end

local TARGET_PLACE_ID = 80692223709267
local isRejoining = false
local isBypassing243 = false

local function autoRejoin()
    if game.PlaceId ~= TARGET_PLACE_ID or isRejoining then return end
    isRejoining = true
    
    local wasFarming = running or wasRunningBeforeDeath
    _G.ScriptRunning = nil
    
    if queue_on_teleport then
        queue_on_teleport([[
            if game.PlaceId == 80692223709267 then
                _G.IsAutoRejoin = true
                _G.WasFarmingBeforeRejoin = ]] .. tostring(wasFarming) .. [[
                repeat task.wait() until game:IsLoaded()
                local p = game:GetService("Players").LocalPlayer
                if not p.Character then p.CharacterAdded:Wait() end
                repeat task.wait() until p and p:FindFirstChild("PlayerGui")
                loadstring(game:HttpGet("https://raw.githubusercontent.com/vovanxuat12345-tech/Obby-for-ugc-AFK-OR-PLAY-/refs/heads/main/README.md?t=" .. os.time()))()
            end
        ]])
    end
    
    task.wait(0.5)
    task.spawn(function()
        if #Players:GetPlayers() <= 1 then
            TeleportService:Teleport(TARGET_PLACE_ID, player)
        else
            TeleportService:TeleportToPlaceInstance(TARGET_PLACE_ID, game.JobId, player)
        end
    end)

    task.delay(12, function()
        if isRejoining and game.PlaceId == TARGET_PLACE_ID then
            TeleportService:Teleport(TARGET_PLACE_ID, player)
        end
    end)
end

GuiService.ErrorMessageChanged:Connect(function()
    if game.PlaceId == TARGET_PLACE_ID then autoRejoin() end
end)

game:GetService("CoreGui").RobloxPromptGui.promptOverlay.ChildAdded:Connect(function(child)
    if child.Name == "ErrorPrompt" and game.PlaceId == TARGET_PLACE_ID then autoRejoin() end
end)

TeleportService.TeleportInitFailed:Connect(function()
    if game.PlaceId == TARGET_PLACE_ID then
        isRejoining = false
        task.wait(1)
        autoRejoin()
    end
end)

local DEFAULT_SETTINGS = {
    FLY_SPEED = 40,
    CHECKPOINT_FOLDER = workspace:WaitForChild("Checkpoints", 15)
}

local currentFlySpeed = DEFAULT_SETTINGS.FLY_SPEED
local running = false
local wasRunningBeforeDeath = false
local isMinimized = false
local isSettingsOpen = false
local espEnabled = true
local playerHighlightEnabled = false
local afkModeEnabled = false

-- CÁC TÍNH NĂNG MẶC ĐỊNH BẬT SẴN (ON)
local autoRebirthEnabled = true 
local antiAfkEnabled = true
local freeCamEnabled = false

local stage251ReachedTime = 0
local lastJumpTime = tick()
local protectionConnection = nil
local currentTargetPart = nil
local farmingTask = nil 

-- FREE CAM ENGINE VARIABLES
local freeCamPos = Vector3.new()
local freeCamYaw = 0   
local freeCamPitch = 0 
local freeCamSpeed = 50
local savedWalkSpeed = 16
local savedJumpPower = 50

local afkStartTime = 0
local afkStartStage = 0
local stagesGained = 0
local lastNotifiedMilestone = -1

local function sendNotification(title, text)
    pcall(function()
        StarterGui:SetCore("SendNotification", {
            Title = title,
            Text = text,
            Duration = 4
        })
    end)
end

local function triggerRebirth()
    pcall(function()
        local mainRemote = ReplicatedStorage:FindFirstChild("MainRemote")
        if mainRemote then
            mainRemote:FireServer({
                Request = "RequestRebirth"
            })
        end
    end)
end

local function lockPlayerCharacter(lock)
    pcall(function()
        local char = player.Character
        local hum = char and char:FindFirstChildOfClass("Humanoid")
        if hum then
            if lock then
                savedWalkSpeed = hum.WalkSpeed > 0 and hum.WalkSpeed or 16
                savedJumpPower = hum.JumpPower > 0 and hum.JumpPower or 50
                hum.WalkSpeed = 0
                hum.JumpPower = 0
                hum.JumpHeight = 0
            else
                hum.WalkSpeed = savedWalkSpeed
                hum.JumpPower = savedJumpPower
                hum.JumpHeight = savedJumpPower
            end
        end
    end)
end

-- ESP 2D SCREENGUI & DRAWING LIBRARY
local espGui = game:GetService("CoreGui"):FindFirstChild("Classic_Stage_ESP") or Instance.new("ScreenGui", game:GetService("CoreGui"))
espGui.Name = "Classic_Stage_ESP"
espGui:ClearAllChildren()
local activeESP = {}

local function createClassicESP(part)
    if not part or not part:IsA("BasePart") or activeESP[part] then return end
    
    local bgui = Instance.new("BillboardGui")
    bgui.Name = "ESP_Text_" .. part.Name; bgui.Adornee = part; bgui.Size = UDim2.new(0, 80, 0, 20); bgui.AlwaysOnTop = true; bgui.ExtentsOffset = Vector3.new(0, 1.8, 0); bgui.Parent = espGui
    local txt = Instance.new("TextLabel")
    txt.Size = UDim2.new(1, 0, 1, 0); txt.BackgroundTransparency = 0.4; txt.BackgroundColor3 = Color3.fromRGB(0, 0, 0); txt.Text = "Stage " .. part.Name; txt.TextColor3 = Color3.fromRGB(255, 50, 50); txt.Font = Enum.Font.SourceSansBold; txt.TextSize = 11; txt.Parent = bgui
    Instance.new("UICorner", txt).CornerRadius = UDim.new(0, 3)
    
    local hl = Instance.new("Highlight")
    hl.Name = "ESP_HL_" .. part.Name; hl.Adornee = part; hl.FillColor = Color3.fromRGB(255, 0, 0); hl.FillTransparency = 0.5; hl.OutlineColor = Color3.fromRGB(255, 255, 255); hl.OutlineTransparency = 0; hl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop; hl.Parent = espGui
    
    local line2d = Drawing.new("Line")
    line2d.Color = Color3.fromRGB(255, 0, 0)
    line2d.Thickness = 1
    line2d.Transparency = 1
    line2d.Visible = false
    
    activeESP[part] = {Text = bgui, Label = txt, Highlight = hl, Line2D = line2d}
end

local function removeClassicESP(part)
    if activeESP[part] then
        if activeESP[part].Text then activeESP[part].Text:Destroy() end
        if activeESP[part].Highlight then activeESP[part].Highlight:Destroy() end
        if activeESP[part].Line2D then 
            pcall(function() activeESP[part].Line2D:Remove() end)
        end
        activeESP[part] = nil
    end
end

local playerHLObj = nil

local function updatePlayerHighlight()
    local char = player.Character
    if not playerHighlightEnabled or not char or afkModeEnabled then
        if playerHLObj then playerHLObj:Destroy(); playerHLObj = nil end
        return
    end

    if not playerHLObj or playerHLObj.Parent ~= espGui then
        if playerHLObj then playerHLObj:Destroy() end
        playerHLObj = Instance.new("Highlight")
        playerHLObj.Name = "Player_Self_Outline"
        playerHLObj.Adornee = char
        playerHLObj.FillTransparency = 1
        playerHLObj.OutlineColor = Color3.fromRGB(255, 0, 0)
        playerHLObj.OutlineTransparency = 0
        playerHLObj.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
        playerHLObj.Parent = espGui
    end
end

-- CHỈ CHO PHÉP XOAY CAMERA Ở NỬA MẢNH MÀN HÌNH BÊN PHẢI (TRÁNH JOYSTICK BÊN TRÁI)
local isRightDragging = false

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if freeCamEnabled and input.UserInputType == Enum.UserInputType.MouseButton2 then
        isRightDragging = true
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton2 then
        isRightDragging = false
    end
end)

UserInputService.InputChanged:Connect(function(input, gameProcessed)
    if not freeCamEnabled then return end

    if input.UserInputType == Enum.UserInputType.MouseMovement and isRightDragging then
        local delta = input.Delta
        freeCamYaw = freeCamYaw - (delta.X * 0.005)
        freeCamPitch = math.clamp(freeCamPitch - (delta.Y * 0.005), math.rad(-80), math.rad(80))
    elseif input.UserInputType == Enum.UserInputType.Touch then
        -- CHỈ NHẬN DIỆN NẾU CHẠM Ở NỬA BÊN PHẢI MÀN HÌNH (X >= ViewportSize.X / 2)
        if input.Position.X >= (camera.ViewportSize.X / 2) then
            local delta = input.Delta
            freeCamYaw = freeCamYaw - (delta.X * 0.005)
            freeCamPitch = math.clamp(freeCamPitch - (delta.Y * 0.005), math.rad(-80), math.rad(80))
        end
    end
end)

RunService.RenderStepped:Connect(function(dt)
    updatePlayerHighlight()

    if freeCamEnabled then
        camera.CameraType = Enum.CameraType.Scriptable
        
        local camRot = CFrame.Angles(0, freeCamYaw, 0) * CFrame.Angles(freeCamPitch, 0, 0)
        
        local moveVector = Vector3.new()
        pcall(function()
            local controls = require(player.PlayerScripts:WaitForChild("PlayerModule")):GetControls()
            moveVector = controls:GetMoveVector()
        end)

        if moveVector.Magnitude > 0 then
            local worldMove = camRot:VectorToWorldSpace(Vector3.new(moveVector.X, 0, moveVector.Z))
            freeCamPos = freeCamPos + (worldMove * freeCamSpeed * dt)
        end

        pcall(function()
            local hum = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
            if hum and hum.WalkSpeed ~= 0 then hum.WalkSpeed = 0 end
        end)

        camera.CFrame = CFrame.new(freeCamPos) * camRot
    else
        if camera.CameraType == Enum.CameraType.Scriptable and not afkModeEnabled then
            camera.CameraType = Enum.CameraType.Custom
        end
    end

    if not espEnabled or afkModeEnabled then
        for part in pairs(activeESP) do removeClassicESP(part) end
        return
    end

    local hrp = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
    if not hrp or not DEFAULT_SETTINGS.CHECKPOINT_FOLDER then return end

    local closestStageNum = -1
    local minDist = math.huge
    for _, cp in pairs(DEFAULT_SETTINGS.CHECKPOINT_FOLDER:GetChildren()) do
        local stageNum = tonumber(cp.Name)
        if stageNum and cp:IsA("BasePart") then
            local d = (hrp.Position - cp.Position).Magnitude
            if d < minDist then minDist = d; closestStageNum = stageNum end
        end
    end

    local validTargets = {}
    if currentTargetPart and currentTargetPart:IsA("BasePart") then
        validTargets[currentTargetPart] = true
        local currentNum = tonumber(currentTargetPart.Name)
        if currentNum then
            local nextPart = DEFAULT_SETTINGS.CHECKPOINT_FOLDER:FindFirstChild(tostring(currentNum + 1))
            if nextPart then validTargets[nextPart] = true end
        end
    else
        local target1 = DEFAULT_SETTINGS.CHECKPOINT_FOLDER:FindFirstChild(tostring(closestStageNum + 1))
        local target2 = DEFAULT_SETTINGS.CHECKPOINT_FOLDER:FindFirstChild(tostring(closestStageNum + 2))
        if target1 then validTargets[target1] = true end
        if target2 then validTargets[target2] = true end
    end

    for part in pairs(activeESP) do if not validTargets[part] then removeClassicESP(part) end end
    for part in pairs(validTargets) do
        if not activeESP[part] then createClassicESP(part) end
        local espData = activeESP[part]
        if espData then
            local basePos = hrp.Position
            if freeCamEnabled then
                basePos = freeCamPos
            end
            local dist = math.floor((basePos - part.Position).Magnitude)
            if espData.Label then espData.Label.Text = "Stage " .. part.Name .. " [" .. dist .. "m]" end
            
            if espData.Line2D then
                local fromVector2D
                if freeCamEnabled then
                    fromVector2D = Vector2.new(camera.ViewportSize.X / 2, camera.ViewportSize.Y / 2)
                else
                    local playerHead = player.Character:FindFirstChild("Head") or hrp
                    local p1 = camera:WorldToViewportPoint(playerHead.Position)
                    fromVector2D = Vector2.new(p1.X, p1.Y)
                end

                local p2 = camera:WorldToViewportPoint(part.Position)
                local toPos = Vector2.new(p2.X, p2.Y)
                if p2.Z < 0 then
                    local screenCenter = Vector2.new(camera.ViewportSize.X / 2, camera.ViewportSize.Y / 2)
                    toPos = screenCenter - (toPos - screenCenter).Unit * 1000
                end

                espData.Line2D.From = fromVector2D
                espData.Line2D.To = toPos
                espData.Line2D.Visible = true
            end
        end
    end
end)

local function createLoadingScreen()
    if not player:FindFirstChild("PlayerGui") then repeat task.wait() until player:FindFirstChild("PlayerGui") end
    local loadingGui = Instance.new("ScreenGui", player.PlayerGui); loadingGui.Name = "Loading_ChienDo"; loadingGui.DisplayOrder = 999
    local bg = Instance.new("Frame", loadingGui); bg.Size = UDim2.new(1, 0, 1, 0); bg.BackgroundColor3 = Color3.new(0, 0, 0); bg.BackgroundTransparency = 0.2; bg.BorderSizePixel = 0
    local centerContainer = Instance.new("Frame", bg); centerContainer.Size = UDim2.new(0, 600, 0, 100); centerContainer.Position = UDim2.new(0.5, -300, 0.5, -50); centerContainer.BackgroundTransparency = 1
    local textContainer = Instance.new("Frame", centerContainer); textContainer.Size = UDim2.new(1, 0, 0, 60); textContainer.Position = UDim2.new(0, 0, 0, 0); textContainer.BackgroundTransparency = 1; textContainer.ClipsDescendants = false 
    local layout = Instance.new("UIListLayout", textContainer); layout.FillDirection = Enum.FillDirection.Horizontal; layout.HorizontalAlignment = Enum.HorizontalAlignment.Center; layout.VerticalAlignment = Enum.VerticalAlignment.Center; layout.SortOrder = Enum.SortOrder.LayoutOrder
    local subVersionLabel = Instance.new("TextLabel", centerContainer); subVersionLabel.Size = UDim2.new(1, 0, 0, 30); subVersionLabel.Position = UDim2.new(0, 0, 0, 65); subVersionLabel.BackgroundTransparency = 1; subVersionLabel.Text = "Owner TikTok: chiendo_not_chiendo"; subVersionLabel.TextColor3 = Color3.fromRGB(180, 180, 180); subVersionLabel.Font = Enum.Font.Gotham; subVersionLabel.TextSize = 16; subVersionLabel.TextTransparency = 1 

    task.spawn(function()
        for i = 1, 120 do
            task.spawn(function()
                local dot = Instance.new("Frame", bg); dot.Size = UDim2.new(0, 5, 0, 5); dot.BackgroundColor3 = Color3.new(1, 0, 0); dot.BorderSizePixel = 0; dot.Position = UDim2.new(math.random(), 0, math.random(), 0); Instance.new("UICorner", dot)
                TweenService:Create(dot, TweenInfo.new(3, Enum.EasingStyle.Linear), {Position = UDim2.new(math.random(), 0, math.random(), 0), BackgroundTransparency = 1}):Play()
            end)
        end
    end)

    local textStr = "OBBY FOR UGC"
    local labels = {}
    local order = 1
    for _, c in utf8.codes(textStr) do
        local char = utf8.char(c)
        local charLabel = Instance.new("TextLabel", textContainer)
        charLabel.Size = UDim2.new(0, char == " " and 12 or 26, 1, 0); charLabel.BackgroundTransparency = 1; charLabel.Text = char; charLabel.TextColor3 = Color3.new(1, 0, 0); charLabel.Font = Enum.Font.GothamBold; charLabel.TextSize = 36; charLabel.LayoutOrder = order; charLabel.TextTransparency = 1; charLabel.Position = UDim2.new(0, 0, 0, 60)
        table.insert(labels, charLabel)
        order = order + 1
    end

    task.spawn(function()
        for index, label in ipairs(labels) do TweenService:Create(label, TweenInfo.new(0.4, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Position = UDim2.new(0, 0, 0, 0), TextTransparency = 0}):Play(); task.wait(0.08) end
        TweenService:Create(subVersionLabel, TweenInfo.new(0.5, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {TextTransparency = 0}):Play()
    end)
    task.wait(3); loadingGui:Destroy()
end

local function makeDraggable(gui)
    local dragging, dragInput, dragStart, startPos
    gui.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then dragging = true; dragStart = input.Position; startPos = gui.Position end
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

local afkGui = Instance.new("ScreenGui", player.PlayerGui)
afkGui.Name = "AFK_FullScreen_Saver"
afkGui.ResetOnSpawn = false
afkGui.DisplayOrder = 998
afkGui.IgnoreGuiInset = true 

local afkScreen = Instance.new("Frame", afkGui)
afkScreen.Size = UDim2.new(1, 0, 1, 0)
afkScreen.BackgroundColor3 = Color3.fromRGB(5, 5, 5)
afkScreen.BorderSizePixel = 0
afkScreen.Visible = false

local statsCard = Instance.new("Frame", afkScreen)
statsCard.Size = UDim2.new(0, 280, 0, 190)
statsCard.Position = UDim2.new(0.5, -140, 0.5, -95)
statsCard.BackgroundColor3 = Color3.fromRGB(15, 15, 15)
statsCard.BorderSizePixel = 0
Instance.new("UICorner", statsCard).CornerRadius = UDim.new(0, 10)
local statsStroke = Instance.new("UIStroke", statsCard)
statsStroke.Color = Color3.fromRGB(255, 0, 0)
statsStroke.Thickness = 1.5

local cardTitle = Instance.new("TextLabel", statsCard)
cardTitle.Size = UDim2.new(1, 0, 0, 30); cardTitle.Position = UDim2.new(0, 0, 0, 8); cardTitle.BackgroundTransparency = 1; cardTitle.Text = "🌙 AFK DASHBOARD"; cardTitle.TextColor3 = Color3.fromRGB(255, 60, 60); cardTitle.Font = Enum.Font.GothamBold; cardTitle.TextSize = 14

local timeLabel = Instance.new("TextLabel", statsCard)
timeLabel.Size = UDim2.new(1, -20, 0, 22); timeLabel.Position = UDim2.new(0, 15, 0, 42); timeLabel.BackgroundTransparency = 1; timeLabel.Text = "⏱️ Time AFK: 00h 00m 00s"; timeLabel.TextColor3 = Color3.fromRGB(220, 220, 220); timeLabel.Font = Enum.Font.GothamMedium; timeLabel.TextSize = 12; timeLabel.TextXAlignment = Enum.TextXAlignment.Left

local stageLabel = Instance.new("TextLabel", statsCard)
stageLabel.Size = UDim2.new(1, -20, 0, 22); stageLabel.Position = UDim2.new(0, 15, 0, 68); stageLabel.BackgroundTransparency = 1; stageLabel.Text = "📍 Current Stage: 0"; stageLabel.TextColor3 = Color3.fromRGB(220, 220, 220); stageLabel.Font = Enum.Font.GothamMedium; stageLabel.TextSize = 12; stageLabel.TextXAlignment = Enum.TextXAlignment.Left

local gainedLabel = Instance.new("TextLabel", statsCard)
gainedLabel.Size = UDim2.new(1, -20, 0, 22); gainedLabel.Position = UDim2.new(0, 15, 0, 94); gainedLabel.BackgroundTransparency = 1; gainedLabel.Text = "📈 Stages Farmed: +0"; gainedLabel.TextColor3 = Color3.fromRGB(0, 255, 120); gainedLabel.Font = Enum.Font.GothamMedium; gainedLabel.TextSize = 12; gainedLabel.TextXAlignment = Enum.TextXAlignment.Left

local scoreLabel = Instance.new("TextLabel", statsCard)
scoreLabel.Size = UDim2.new(1, -20, 0, 22); scoreLabel.Position = UDim2.new(0, 15, 0, 120); scoreLabel.BackgroundTransparency = 1; scoreLabel.Text = "🏆 Current Score: 0"; scoreLabel.TextColor3 = Color3.fromRGB(255, 200, 0); scoreLabel.Font = Enum.Font.GothamMedium; scoreLabel.TextSize = 12; scoreLabel.TextXAlignment = Enum.TextXAlignment.Left

local exitAfkBtn = Instance.new("TextButton", statsCard)
exitAfkBtn.Size = UDim2.new(1, -30, 0, 28); exitAfkBtn.Position = UDim2.new(0, 15, 0, 150); exitAfkBtn.BackgroundColor3 = Color3.fromRGB(255, 40, 40); exitAfkBtn.Text = "EXIT AFK MODE"; exitAfkBtn.TextColor3 = Color3.new(1, 1, 1); exitAfkBtn.Font = Enum.Font.GothamBold; exitAfkBtn.TextSize = 11
Instance.new("UICorner", exitAfkBtn).CornerRadius = UDim.new(0, 6)

local function getCurrentStageNum()
    local hrp = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
    if not hrp or not DEFAULT_SETTINGS.CHECKPOINT_FOLDER then return 0 end
    local closest = 0
    local minDist = math.huge
    for _, cp in pairs(DEFAULT_SETTINGS.CHECKPOINT_FOLDER:GetChildren()) do
        local stageNum = tonumber(cp.Name)
        if stageNum and cp:IsA("BasePart") then
            local d = (hrp.Position - cp.Position).Magnitude
            if d < minDist then minDist = d; closest = stageNum end
        end
    end
    return closest
end

local function getLeaderstatScore()
    local ls = player:FindFirstChild("leaderstats")
    if ls then
        local scoreObj = ls:FindFirstChild("Stage") or ls:FindFirstChild("Score") or ls:FindFirstChild("Points") or ls:FindFirstChild("UGC")
        if scoreObj then return scoreObj.Value end
    end
    return 0
end

task.spawn(function()
    while afkGui.Parent do
        local currentStage = getCurrentStageNum()
        if running and currentStage > 0 and currentStage % 50 == 0 and currentStage ~= lastNotifiedMilestone then
            lastNotifiedMilestone = currentStage
            sendNotification("🎉 STAGE MILESTONE!", "Chúc mừng! Bạn đã đạt Stage " .. tostring(currentStage) .. "!")
        end

        if autoRebirthEnabled and running then
            if currentStage == 251 then
                if stage251ReachedTime == 0 then
                    stage251ReachedTime = tick()
                elseif tick() - stage251ReachedTime >= 10 then
                    triggerRebirth()
                    stage251ReachedTime = tick()
                end
            else
                stage251ReachedTime = 0
            end
        else
            stage251ReachedTime = 0
        end

        if antiAfkEnabled then
            if tick() - lastJumpTime >= 900 then
                lastJumpTime = tick()
                pcall(function()
                    local hum = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
                    if hum then
                        hum.Jump = true
                    else
                        VirtualUser:Button1Down(Vector2.new(0,0), workspace.CurrentCamera.CFrame)
                        VirtualUser:Button1Up(Vector2.new(0,0), workspace.CurrentCamera.CFrame)
                    end
                end)
            end
        end

        if afkModeEnabled then
            local elapsed = math.floor(tick() - afkStartTime)
            local hrs = math.floor(elapsed / 3600)
            local mins = math.floor((elapsed % 3600) / 60)
            local secs = elapsed % 60
            timeLabel.Text = string.format("⏱️ Time AFK: %02dh %02dm %02ds", hrs, mins, secs)
            
            stageLabel.Text = "📍 Current Stage: " .. tostring(currentStage)
            
            if currentStage >= afkStartStage then
                stagesGained = currentStage - afkStartStage
            end
            gainedLabel.Text = "📈 Stages Farmed: +" .. tostring(stagesGained)
            scoreLabel.Text = "🏆 Current Score: " .. tostring(getLeaderstatScore())
        end
        task.wait(1)
    end
end)

local mainGui = Instance.new("ScreenGui", player.PlayerGui)
mainGui.Name = "AutoObby_GodAlways"
mainGui.ResetOnSpawn = false
mainGui.DisplayOrder = 999

local frame = Instance.new("Frame", mainGui)
frame.Size = UDim2.new(0, 220, 0, 180); frame.Position = UDim2.new(0, 30, 0.5, -90); frame.BackgroundColor3 = Color3.fromRGB(15, 15, 15); frame.BorderSizePixel = 0; frame.ClipsDescendants = false; frame.Active = true
Instance.new("UICorner", frame); makeDraggable(frame)

local settingsPanel = Instance.new("Frame", frame)
settingsPanel.Size = UDim2.new(0, 180, 0, 226); settingsPanel.Position = UDim2.new(1, 8, 0, 0); settingsPanel.BackgroundColor3 = Color3.fromRGB(15, 15, 15); settingsPanel.BorderSizePixel = 0; settingsPanel.Visible = false; settingsPanel.ZIndex = 4; settingsPanel.ClipsDescendants = true
Instance.new("UICorner", settingsPanel)

task.spawn(function()
    while mainGui.Parent do
        if settingsPanel.Visible and not afkModeEnabled then
            local dot = Instance.new("Frame", settingsPanel); dot.Size = UDim2.new(0, 3, 0, 3); dot.BackgroundColor3 = Color3.new(1, 0, 0); dot.Position = UDim2.new(math.random(), 0, math.random(), 0); dot.ZIndex = 1; dot.BorderSizePixel = 0; Instance.new("UICorner", dot)
            TweenService:Create(dot, TweenInfo.new(2, Enum.EasingStyle.Linear), {Position = UDim2.new(math.random(), 0, math.random(), 0), BackgroundTransparency = 1}):Play()
            game:GetService("Debris"):AddItem(dot, 2)
        end
        task.wait(0.1)
    end
end)

local settingsTitle = Instance.new("TextLabel", settingsPanel)
settingsTitle.Size = UDim2.new(1, 0, 0, 24); settingsTitle.Position = UDim2.new(0, 0, 0, 4); settingsTitle.BackgroundTransparency = 1; settingsTitle.Text = "SETTINGS [v1.107]"; settingsTitle.TextColor3 = Color3.new(1, 1, 1); settingsTitle.Font = Enum.Font.GothamBold; settingsTitle.TextSize = 12; settingsTitle.ZIndex = 6

-- 1. FLY SPEED CONTROL
local speedTitle = Instance.new("TextLabel", settingsPanel)
speedTitle.Size = UDim2.new(1, -20, 0, 16); speedTitle.Position = UDim2.new(0, 8, 0, 26); speedTitle.BackgroundTransparency = 1; speedTitle.Text = "Fly Speed (20 - 70):"; speedTitle.TextColor3 = Color3.fromRGB(220, 220, 220); speedTitle.Font = Enum.Font.Gotham; speedTitle.TextSize = 10; speedTitle.TextXAlignment = Enum.TextXAlignment.Left; speedTitle.ZIndex = 6

local speedInput = Instance.new("TextBox", settingsPanel)
speedInput.Size = UDim2.new(0, 40, 0, 18); speedInput.Position = UDim2.new(1, -48, 0, 24); speedInput.BackgroundColor3 = Color3.fromRGB(30, 30, 30); speedInput.Text = tostring(currentFlySpeed); speedInput.TextColor3 = Color3.new(1, 0, 0); speedInput.Font = Enum.Font.GothamBold; speedInput.TextSize = 10; speedInput.ZIndex = 7; speedInput.ClearTextOnFocus = false
Instance.new("UICorner", speedInput).CornerRadius = UDim.new(0, 4)

local sliderBG = Instance.new("Frame", settingsPanel)
sliderBG.Size = UDim2.new(1, -16, 0, 5); sliderBG.Position = UDim2.new(0, 8, 0, 46); sliderBG.BackgroundColor3 = Color3.fromRGB(35, 35, 35); sliderBG.BorderSizePixel = 0; sliderBG.ZIndex = 6
Instance.new("UICorner", sliderBG).CornerRadius = UDim.new(0, 4)

local sliderFill = Instance.new("Frame", sliderBG)
local initialScale = (currentFlySpeed - 20) / (70 - 20)
sliderFill.Size = UDim2.new(math.clamp(initialScale, 0, 1), 0, 1, 0); sliderFill.BackgroundColor3 = Color3.new(1, 0, 0); sliderFill.BorderSizePixel = 0; sliderFill.ZIndex = 7
Instance.new("UICorner", sliderFill).CornerRadius = UDim.new(0, 4)

local sliderBtn = Instance.new("TextButton", sliderBG)
sliderBtn.Size = UDim2.new(1, 0, 1, 0); sliderBtn.BackgroundTransparency = 1; sliderBtn.Text = ""; sliderBtn.ZIndex = 8

local function updateSpeed(val)
    val = math.clamp(math.floor(val + 0.5), 20, 70)
    currentFlySpeed = val
    speedInput.Text = tostring(val)
    local scale = (val - 20) / (70 - 20)
    sliderFill.Size = UDim2.new(scale, 0, 1, 0)
end

speedInput.FocusLost:Connect(function()
    local num = tonumber(speedInput.Text)
    if num then updateSpeed(num) else speedInput.Text = tostring(currentFlySpeed) end
end)

local draggingSlider = false
sliderBtn.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        draggingSlider = true
        local absPos = sliderBG.AbsolutePosition.X
        local absSize = sliderBG.AbsoluteSize.X
        local mousePos = input.Position.X
        local rel = math.clamp((mousePos - absPos) / absSize, 0, 1)
        updateSpeed(20 + rel * (70 - 20))
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if draggingSlider and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
        local absPos = sliderBG.AbsolutePosition.X
        local absSize = sliderBG.AbsoluteSize.X
        local mousePos = input.Position.X
        local rel = math.clamp((mousePos - absPos) / absSize, 0, 1)
        updateSpeed(20 + rel * (70 - 20))
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        draggingSlider = false
    end
end)

-- 2. ESP BUTTON
local espBtn = Instance.new("TextButton", settingsPanel)
espBtn.Size = UDim2.new(1, -16, 0, 20); espBtn.Position = UDim2.new(0, 8, 0, 58); espBtn.BackgroundTransparency = 1; espBtn.Text = "🔴 ESP Lines"; espBtn.TextColor3 = Color3.fromRGB(220, 220, 220); espBtn.Font = Enum.Font.Gotham; espBtn.TextSize = 10; espBtn.TextXAlignment = Enum.TextXAlignment.Left; espBtn.ZIndex = 7

local function updateEspUI()
    if espEnabled then
        espBtn.Text = "🔴 ESP Lines"
    else
        espBtn.Text = "ESP Lines  OFF"
    end
end

espBtn.MouseButton1Click:Connect(function()
    espEnabled = not espEnabled
    updateEspUI()
end)

-- 3. PLAYER HIGHLIGHT BUTTON
local playerHlBtn = Instance.new("TextButton", settingsPanel)
playerHlBtn.Size = UDim2.new(1, -16, 0, 20); playerHlBtn.Position = UDim2.new(0, 8, 0, 80); playerHlBtn.BackgroundTransparency = 1; playerHlBtn.Text = "Player Highlight  OFF"; playerHlBtn.TextColor3 = Color3.fromRGB(220, 220, 220); playerHlBtn.Font = Enum.Font.Gotham; playerHlBtn.TextSize = 10; playerHlBtn.TextXAlignment = Enum.TextXAlignment.Left; playerHlBtn.ZIndex = 7

local function updatePlayerHlUI()
    if playerHighlightEnabled then
        playerHlBtn.Text = "🔴 Player Highlight"
    else
        playerHlBtn.Text = "Player Highlight  OFF"
    end
end

playerHlBtn.MouseButton1Click:Connect(function()
    playerHighlightEnabled = not playerHighlightEnabled
    updatePlayerHlUI()
    updatePlayerHighlight()
end)

-- 4. AUTO REBIRTH BUTTON
local rebirthBtn = Instance.new("TextButton", settingsPanel)
rebirthBtn.Size = UDim2.new(1, -16, 0, 20); rebirthBtn.Position = UDim2.new(0, 8, 0, 102); rebirthBtn.BackgroundTransparency = 1; rebirthBtn.Text = "🔴 Auto Rebirth"; rebirthBtn.TextColor3 = Color3.fromRGB(220, 220, 220); rebirthBtn.Font = Enum.Font.Gotham; rebirthBtn.TextSize = 10; rebirthBtn.TextXAlignment = Enum.TextXAlignment.Left; rebirthBtn.ZIndex = 7

local function updateRebirthUI()
    if autoRebirthEnabled then
        rebirthBtn.Text = "🔴 Auto Rebirth"
    else
        rebirthBtn.Text = "Auto Rebirth  OFF"
    end
end

rebirthBtn.MouseButton1Click:Connect(function()
    autoRebirthEnabled = not autoRebirthEnabled
    updateRebirthUI()
end)

-- 5. ANTI-AFK BUTTON
local antiAfkBtn = Instance.new("TextButton", settingsPanel)
antiAfkBtn.Size = UDim2.new(1, -16, 0, 20); antiAfkBtn.Position = UDim2.new(0, 8, 0, 124); antiAfkBtn.BackgroundTransparency = 1; antiAfkBtn.Text = "🔴 Anti-AFK (15m)"; antiAfkBtn.TextColor3 = Color3.fromRGB(220, 220, 220); antiAfkBtn.Font = Enum.Font.Gotham; antiAfkBtn.TextSize = 10; antiAfkBtn.TextXAlignment = Enum.TextXAlignment.Left; antiAfkBtn.ZIndex = 7

local function updateAntiAfkUI()
    if antiAfkEnabled then
        antiAfkBtn.Text = "🔴 Anti-AFK (15m)"
    else
        antiAfkBtn.Text = "Anti-AFK  OFF"
    end
end

antiAfkBtn.MouseButton1Click:Connect(function()
    antiAfkEnabled = not antiAfkEnabled
    updateAntiAfkUI()
end)

-- 6. FREE CAM BUTTON
local freeCamBtn = Instance.new("TextButton", settingsPanel)
freeCamBtn.Size = UDim2.new(1, -16, 0, 20); freeCamBtn.Position = UDim2.new(0, 8, 0, 146); freeCamBtn.BackgroundTransparency = 1; freeCamBtn.Text = "Free Cam  OFF"; freeCamBtn.TextColor3 = Color3.fromRGB(220, 220, 220); freeCamBtn.Font = Enum.Font.Gotham; freeCamBtn.TextSize = 10; freeCamBtn.TextXAlignment = Enum.TextXAlignment.Left; freeCamBtn.ZIndex = 7

local function toggleFreeCam(state)
    freeCamEnabled = state
    lockPlayerCharacter(freeCamEnabled)
    
    if freeCamEnabled then
        freeCamBtn.Text = "🔴 Free Cam"
        freeCamPos = camera.CFrame.Position
        local _, y, _ = camera.CFrame:ToOrientation()
        freeCamYaw = y
        freeCamPitch = 0
    else
        freeCamBtn.Text = "Free Cam  OFF"
        camera.CameraType = Enum.CameraType.Custom
    end
end

freeCamBtn.MouseButton1Click:Connect(function()
    toggleFreeCam(not freeCamEnabled)
end)

-- 7. AFK MODE BUTTON
local afkBtn = Instance.new("TextButton", settingsPanel)
afkBtn.Size = UDim2.new(1, -16, 0, 20); afkBtn.Position = UDim2.new(0, 8, 0, 168); afkBtn.BackgroundTransparency = 1; afkBtn.Text = "AFK Mode  OFF"; afkBtn.TextColor3 = Color3.fromRGB(220, 220, 220); afkBtn.Font = Enum.Font.Gotham; afkBtn.TextSize = 10; afkBtn.TextXAlignment = Enum.TextXAlignment.Left; afkBtn.ZIndex = 7

local function toggleAfkMode(state)
    afkModeEnabled = state
    afkScreen.Visible = afkModeEnabled
    
    if afkModeEnabled then
        afkStartTime = tick()
        afkStartStage = getCurrentStageNum()
        stagesGained = 0
        afkBtn.Text = "🔴 AFK Mode"
        pcall(function()
            settings():GetService("RenderSettings").QualityLevel = Enum.QualityLevel.Level01
            if set3drenderingenabled then set3drenderingenabled(false) end
        end)
    else
        afkBtn.Text = "AFK Mode  OFF"
        pcall(function()
            settings():GetService("RenderSettings").QualityLevel = Enum.QualityLevel.Automatic
            if set3drenderingenabled then set3drenderingenabled(true) end
        end)
    end
    updatePlayerHighlight()
end

afkBtn.MouseButton1Click:Connect(function() toggleAfkMode(not afkModeEnabled) end)
exitAfkBtn.MouseButton1Click:Connect(function() toggleAfkMode(false) end)

task.spawn(function()
    while mainGui.Parent do
        if not afkModeEnabled then
            local dot = Instance.new("Frame", frame); dot.Size = UDim2.new(0, 3, 0, 3); dot.BackgroundColor3 = Color3.new(1, 0, 0); dot.Position = UDim2.new(math.random(), 0, math.random(), 0); dot.ZIndex = 1; dot.BorderSizePixel = 0; Instance.new("UICorner", dot)
            TweenService:Create(dot, TweenInfo.new(2, Enum.EasingStyle.Linear), {Position = UDim2.new(math.random(), 0, math.random(), 0), BackgroundTransparency = 1}):Play()
            game:GetService("Debris"):AddItem(dot, 2)
        end
        task.wait(0.1)
    end
end)

local topContent = Instance.new("Frame", frame); topContent.Size = UDim2.new(1, 0, 0, 70); topContent.BackgroundTransparency = 1; topContent.ZIndex = 5
local title = Instance.new("TextLabel", topContent); title.Size = UDim2.new(1, 0, 0, 20); title.Position = UDim2.new(0, 0, 0, 6); title.Text = "OBBY FOR UGC"; title.TextColor3 = Color3.new(1, 1, 1); title.Font = Enum.Font.GothamBold; title.TextSize = 15; title.BackgroundTransparency = 1; title.ZIndex = 6
local subTitle1 = Instance.new("TextLabel", topContent); subTitle1.Size = UDim2.new(1, 0, 0, 16); subTitle1.Position = UDim2.new(0, 0, 0, 24); subTitle1.Text = "(AFK OR PLAY)"; subTitle1.TextColor3 = Color3.fromRGB(200, 200, 200); subTitle1.Font = Enum.Font.Gotham; subTitle1.TextSize = 11; subTitle1.BackgroundTransparency = 1; subTitle1.ZIndex = 6
local subTitle2 = Instance.new("TextLabel", topContent); subTitle2.Size = UDim2.new(1, 0, 0, 26); subTitle2.Position = UDim2.new(0, 0, 0, 38); subTitle2.Text = "Owner\nTiktok : Chiendo_not_chiendo"; subTitle2.TextColor3 = Color3.fromRGB(160, 160, 160); subTitle2.Font = Enum.Font.Gotham; subTitle2.TextSize = 10; subTitle2.BackgroundTransparency = 1; subTitle2.ZIndex = 6

local settingsBtn = Instance.new("TextButton", frame); settingsBtn.Size = UDim2.new(0, 25, 0, 25); settingsBtn.Position = UDim2.new(0, 5, 0, 5); settingsBtn.Text = "⚙️"; settingsBtn.TextColor3 = Color3.new(1, 1, 1); settingsBtn.BackgroundTransparency = 1; settingsBtn.Font = Enum.Font.GothamBold; settingsBtn.TextSize = 16; settingsBtn.ZIndex = 10
local infoBtn = Instance.new("TextButton", frame); infoBtn.Size = UDim2.new(0, 25, 0, 25); infoBtn.Position = UDim2.new(0, 32, 0, 5); infoBtn.Text = "!"; infoBtn.TextColor3 = Color3.new(1, 0, 0); infoBtn.BackgroundTransparency = 1; infoBtn.Font = Enum.Font.GothamBold; infoBtn.TextSize = 20; infoBtn.ZIndex = 10

settingsBtn.MouseButton1Click:Connect(function()
    isSettingsOpen = not isSettingsOpen
    settingsPanel.Visible = isSettingsOpen
end)

local bugReportFrame = Instance.new("TextButton", mainGui); bugReportFrame.Size = UDim2.new(0, 320, 0, 180); bugReportFrame.Position = UDim2.new(0.5, -160, 0.5, -90); bugReportFrame.BackgroundColor3 = Color3.fromRGB(10, 10, 10); bugReportFrame.BackgroundTransparency = 0.25; bugReportFrame.Visible = false; bugReportFrame.ZIndex = 100
Instance.new("UICorner", bugReportFrame).CornerRadius = UDim.new(0, 8)
local stroke = Instance.new("UIStroke", bugReportFrame); stroke.Color = Color3.new(1, 0, 0); stroke.Thickness = 2; stroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
local bugTextLabel = Instance.new("TextLabel", bugReportFrame); bugTextLabel.Size = UDim2.new(1, -20, 1, -20); bugTextLabel.Position = UDim2.new(0, 10, 0, 10); bugTextLabel.BackgroundTransparency = 1; bugTextLabel.Text = "-INFO BOARD-\n-Update 1.107 🚀\n+Split screen touch: left side free for joystick, right side for Free Cam rotation\n+Zero tilt & ultra smooth control"; bugTextLabel.TextColor3 = Color3.new(1, 1, 1); bugTextLabel.Font = Enum.Font.GothamMedium; bugTextLabel.TextSize = 12; bugTextLabel.TextWrapped = true; bugTextLabel.TextXAlignment = Enum.TextXAlignment.Left; bugTextLabel.TextYAlignment = Enum.TextYAlignment.Top; bugTextLabel.ZIndex = 101

infoBtn.MouseButton1Click:Connect(function() bugReportFrame.Visible = not bugReportFrame.Visible end)
bugReportFrame.MouseButton1Click:Connect(function() bugReportFrame.Visible = false end)

local bottomContent = Instance.new("Frame", frame); bottomContent.Size = UDim2.new(1, 0, 0, 110); bottomContent.Position = UDim2.new(0, 0, 0, 70); bottomContent.BackgroundTransparency = 1; bottomContent.ZIndex = 5
local btn = Instance.new("TextButton", bottomContent); btn.Size = UDim2.new(0, 180, 0, 50); btn.Position = UDim2.new(0.5, -90, 0, 20); btn.Text = "Auto Farm Stage: OFF"; btn.BackgroundColor3 = Color3.new(1, 1, 1); btn.TextColor3 = Color3.new(0, 0, 0); btn.Font = Enum.Font.GothamBold; btn.TextSize = 15; btn.ZIndex = 6; btn.Active = true
Instance.new("UICorner", btn)
local toggleBtn = Instance.new("TextButton", frame); toggleBtn.Size = UDim2.new(0, 25, 0, 25); toggleBtn.Position = UDim2.new(1, -30, 0, 5); toggleBtn.Text = "▲"; toggleBtn.TextColor3 = Color3.new(1, 1, 1); toggleBtn.BackgroundTransparency = 1; toggleBtn.Font = Enum.Font.GothamBold; toggleBtn.TextSize = 18; toggleBtn.ZIndex = 10

toggleBtn.MouseButton1Click:Connect(function()
    isMinimized = not isMinimized
    if isMinimized then
        frame:TweenSize(UDim2.new(0, 220, 0, 70), "Out", "Quart", 0.3, true); toggleBtn.Text = "▼"; bottomContent.Visible = false; settingsPanel.Visible = false
    else
        frame:TweenSize(UDim2.new(0, 220, 0, 180), "Out", "Quart", 0.3, true); toggleBtn.Text = "▲"; bottomContent.Visible = true; settingsPanel.Visible = isSettingsOpen
    end
end)

local function toggleProtection(state)
    if state then
        if not protectionConnection then
            protectionConnection = RunService.Stepped:Connect(function()
                if running and player.Character then
                    local h = player.Character:FindFirstChildOfClass("Humanoid")
                    if h then h.MaxHealth = math.huge; h.Health = math.huge; h:SetStateEnabled(Enum.HumanoidStateType.Dead, false) end
                    for _, p in pairs(player.Character:GetDescendants()) do 
                        if p:IsA("BasePart") then p.CanCollide = false; p.AssemblyLinearVelocity = Vector3.zero; p.AssemblyAngularVelocity = Vector3.zero end 
                    end
                end
            end)
        end
    else
        if protectionConnection then protectionConnection:Disconnect(); protectionConnection = nil end
    end
end

local function handleStage243Bypass(target)
    if isBypassing243 then return end
    isBypassing243 = true

    local hrp = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
    if not hrp or not running then isBypassing243 = false return end

    local targetCFrame = target and (target.CFrame * CFrame.new(0, 1.5, 0)) or CFrame.new(1761, 776.5, -704.5)

    local startCFrame = hrp.CFrame
    local distanceTo243 = (startCFrame.Position - targetCFrame.Position).Magnitude
    local durationTo243 = distanceTo243 / currentFlySpeed
    local startTimeTo243 = tick()

    local moveConn
    moveConn = RunService.RenderStepped:Connect(function()
        if not running or not hrp or not hrp.Parent then if moveConn then moveConn:Disconnect() end return end
        local alpha = math.clamp((tick() - startTimeTo243) / durationTo243, 0, 1)
        hrp.CFrame = startCFrame:Lerp(targetCFrame, alpha)
        if alpha >= 1 then if moveConn then moveConn:Disconnect() end end
    end)

    while running and (tick() - startTimeTo243) < durationTo243 do task.wait() end
    if moveConn then moveConn:Disconnect() end
    if not running then isBypassing243 = false return end

    hrp.CFrame = targetCFrame
    if firetouchinterest and target and target:IsA("BasePart") then
        firetouchinterest(hrp, target, 0); task.wait(0.05); firetouchinterest(hrp, target, 1)
    end
    task.wait(0.2)

    local stage243Pos = hrp.Position
    local offsetCFrame = CFrame.new(stage243Pos.X, stage243Pos.Y, stage243Pos.Z + 200)
    local durationZ = 200 / currentFlySpeed
    
    local startTimeZ1 = tick()
    local connZ1
    connZ1 = RunService.RenderStepped:Connect(function()
        if not running or not hrp or not hrp.Parent then if connZ1 then connZ1:Disconnect() end return end
        local alpha = math.clamp((tick() - startTimeZ1) / durationZ, 0, 1)
        hrp.CFrame = CFrame.new(stage243Pos):Lerp(offsetCFrame, alpha)
        if alpha >= 1 then if connZ1 then connZ1:Disconnect() end end
    end)
    while running and (tick() - startTimeZ1) < durationZ do task.wait() end
    if connZ1 then connZ1:Disconnect() end
    if not running then isBypassing243 = false return end

    hrp.CFrame = offsetCFrame

    toggleProtection(false)
    task.wait(math.random(20, 30) / 10)
    if running then toggleProtection(true) end

    isBypassing243 = false
end

local function flyToTarget(target)
    currentTargetPart = target
    local hrp = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
    if not hrp or not target or not running then currentTargetPart = nil return end
    
    if target.Name == "243" then
        handleStage243Bypass(target)
        currentTargetPart = nil
        return
    end

    local startCFrame = hrp.CFrame
    local targetCFrame = target.CFrame * CFrame.new(0, 2, 0)
    local distance = (startCFrame.Position - targetCFrame.Position).Magnitude
    local duration = distance / currentFlySpeed
    local startTime = tick()
    
    local moveConnection
    moveConnection = RunService.RenderStepped:Connect(function()
        if not running or not hrp or not hrp.Parent then if moveConnection then moveConnection:Disconnect() end return end
        local elapsed = tick() - startTime
        local alpha = math.clamp(elapsed / duration, 0, 1)
        hrp.CFrame = startCFrame:Lerp(targetCFrame, alpha)
        if alpha >= 1 then if moveConnection then moveConnection:Disconnect() end end
    end)
    
    while running and (tick() - startTime) < duration do task.wait() end
    if moveConnection then moveConnection:Disconnect() end
    if not running then currentTargetPart = nil return end

    hrp.CFrame = target.CFrame * CFrame.new(0, 1.5, 0)
    if firetouchinterest and target:IsA("BasePart") then
        firetouchinterest(hrp, target, 0); task.wait(0.05); firetouchinterest(hrp, target, 1)
    end
    
    toggleProtection(false)
    local pauseTime = math.random(3, 5) / 10
    task.wait(pauseTime)
    
    currentTargetPart = nil
    if running then toggleProtection(true) end
end

local function startFarming()
    running = true
    wasRunningBeforeDeath = true
    toggleProtection(true)
    btn.Text = "Auto Farm Stage: ON"; btn.BackgroundColor3 = Color3.new(1, 0, 0); btn.TextColor3 = Color3.new(1, 1, 1)
    
    if farmingTask then return end 
    
    farmingTask = task.spawn(function()
        local hrp = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
        if hrp and DEFAULT_SETTINGS.CHECKPOINT_FOLDER then
            local cp243 = DEFAULT_SETTINGS.CHECKPOINT_FOLDER:FindFirstChild("243")
            local cp242 = DEFAULT_SETTINGS.CHECKPOINT_FOLDER:FindFirstChild("242")
            if cp243 and cp242 then
                local dist243 = (hrp.Position - cp243.Position).Magnitude
                if dist243 <= 25 then
                    local startCFrame = hrp.CFrame
                    local targetCFrame = cp242.CFrame * CFrame.new(0, 1.5, 0)
                    local distance = (startCFrame.Position - targetCFrame.Position).Magnitude
                    local duration = distance / currentFlySpeed
                    local startTime = tick()
                    
                    local moveConn
                    moveConn = RunService.RenderStepped:Connect(function()
                        if not running or not hrp then if moveConn then moveConn:Disconnect() end return end
                        local alpha = math.clamp((tick() - startTime) / duration, 0, 1)
                        hrp.CFrame = startCFrame:Lerp(targetCFrame, alpha)
                        if alpha >= 1 then if moveConn then moveConn:Disconnect() end end
                    end)
                    while running and (tick() - startTime) < duration do task.wait() end
                    if moveConn then moveConn:Disconnect() end

                    if running and hrp then
                        hrp.CFrame = targetCFrame
                        if firetouchinterest and cp242:IsA("BasePart") then
                            firetouchinterest(hrp, cp242, 0); task.wait(0.05); firetouchinterest(hrp, cp242, 1)
                        end
                    end
                    task.wait(math.random(3, 5) / 10)
                end
            end
        end

        local lastStageNum = -1
        local stuckTime = 0
        local isSkippingStuck = false
        
        while running do
            local hrp = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
            if not hrp then task.wait(0.5) continue end
            
            local closestStageNum = -1
            local minDist = math.huge
            if DEFAULT_SETTINGS.CHECKPOINT_FOLDER then
                for _, cp in pairs(DEFAULT_SETTINGS.CHECKPOINT_FOLDER:GetChildren()) do
                    local stageNum = tonumber(cp.Name)
                    if stageNum and cp:IsA("BasePart") then
                        local d = (hrp.Position - cp.Position).Magnitude
                        if d < minDist then minDist = d; closestStageNum = stageNum end
                    end
                end
            end
            
            if closestStageNum == lastStageNum then
                stuckTime = stuckTime + 0.5
                if stuckTime >= 15 and not isSkippingStuck then
                    isSkippingStuck = true
                    local nextStageObj = DEFAULT_SETTINGS.CHECKPOINT_FOLDER and DEFAULT_SETTINGS.CHECKPOINT_FOLDER:FindFirstChild(tostring(closestStageNum + 1))
                    if nextStageObj then flyToTarget(nextStageObj) end
                    stuckTime = 0; isSkippingStuck = false
                    task.wait(0.5)
                    continue
                end
            else
                lastStageNum = closestStageNum
                stuckTime = 0
            end
            
            local nextTarget = DEFAULT_SETTINGS.CHECKPOINT_FOLDER and DEFAULT_SETTINGS.CHECKPOINT_FOLDER:FindFirstChild(tostring(closestStageNum + 1))
            if nextTarget then flyToTarget(nextTarget) else task.wait(0.5) end
        end
        farmingTask = nil
    end)
end

local function stopFarming()
    running = false
    wasRunningBeforeDeath = false
    currentTargetPart = nil
    toggleProtection(false)
    btn.Text = "Auto Farm Stage: OFF"; btn.BackgroundColor3 = Color3.new(1, 1, 1); btn.TextColor3 = Color3.new(0, 0, 0)
end

btn.MouseButton1Click:Connect(function()
    if running then stopFarming() else startFarming() end
end)

player.CharacterAdded:Connect(function(newChar) 
    toggleProtection(false)
    currentTargetPart = nil
    
    if freeCamEnabled then
        lockPlayerCharacter(true)
    end
    
    if wasRunningBeforeDeath then
        task.spawn(function()
            newChar:WaitForChild("HumanoidRootPart", 9e9)
            task.wait(1)
            if wasRunningBeforeDeath then startFarming() end
        end)
    else
        btn.Text = "Auto Farm Stage: OFF"; btn.BackgroundColor3 = Color3.new(1, 1, 1); btn.TextColor3 = Color3.new(0, 0, 0)
    end
end)

if _G.IsAutoRejoin then
    _G.IsAutoRejoin = nil
    task.spawn(function()
        local char = player.Character or player.CharacterAdded:Wait()
        char:WaitForChild("HumanoidRootPart", 9e9)
        task.wait(1.5)
        startFarming()
    end)
end
