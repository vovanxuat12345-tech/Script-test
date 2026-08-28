local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local TeleportService = game:GetService("TeleportService")
local GuiService = game:GetService("GuiService")
local player = Players.LocalPlayer

-- ========================================================
-- KIỂM TRA CHỐNG TRÙNG SCRIPT TOÀN CỤC
-- ========================================================
if _G.ScriptRunning then
    return
end
_G.ScriptRunning = true 

local oldGui = player:WaitForChild("PlayerGui"):FindFirstChild("AutoObby_GodAlways")
if oldGui then oldGui:Destroy() end
local oldLoading = player:WaitForChild("PlayerGui"):FindFirstChild("Loading_ChienDo")
if oldLoading then oldLoading:Destroy() end

-- ==========================================
-- ĐOẠN XỬ LÝ AUTO REJOIN
-- ==========================================
local isRejoining = false

local function autoRejoin()
    if isRejoining then return end
    isRejoining = true
    _G.ScriptRunning = nil
    
    if queue_on_teleport then
        queue_on_teleport([[
            _G.IsAutoRejoin = true
            repeat task.wait() until game:IsLoaded()
            local p = game:GetService("Players").LocalPlayer
            repeat task.wait() until p and p:FindFirstChild("PlayerGui")
            
            loadstring(game:HttpGet("https://raw.githubusercontent.com/vovanxuat12345-tech/Script-test/refs/heads/main/README.md?t=" .. os.time()))()
        ]])
    end
    
    task.wait(0.5)
    if #Players:GetPlayers() <= 1 then
        TeleportService:Teleport(game.PlaceId, player)
    else
        TeleportService:TeleportToPlaceInstance(game.PlaceId, game.JobId, player)
    end
end

GuiService.ErrorMessageChanged:Connect(function()
    autoRejoin()
end)

game:GetService("CoreGui").RobloxPromptGui.promptOverlay.ChildAdded:Connect(function(child)
    if child.Name == "ErrorPrompt" then
        autoRejoin()
    end
end)

-- ==========================================
-- CÀI ĐẶT & BIẾN SỐ
-- ==========================================
local DEFAULT_SETTINGS = {
    FLY_SPEED = 40,
    CHECKPOINT_FOLDER = workspace:WaitForChild("Checkpoints", 15)
}

local currentFlySpeed = DEFAULT_SETTINGS.FLY_SPEED

local running = false
local wasRunningBeforeDeath = false
local isMinimized = false
local protectionConnection = nil
local currentTargetPart = nil

-- ==========================================
-- STAGE ESP (BOX LUÔN HIỆN + TRACER LINE + SỐ MET)
-- ==========================================
local espFolder = game:GetService("CoreGui"):FindFirstChild("Classic_Stage_ESP") or Instance.new("Folder", game:GetService("CoreGui"))
espFolder.Name = "Classic_Stage_ESP"
espFolder:ClearAllChildren()

local activeESP = {}

local function createClassicESP(part)
    if not part or not part:IsA("BasePart") or activeESP[part] then return end

    -- BillboardGui (Box chữ Stage & Số Mét)
    local bgui = Instance.new("BillboardGui")
    bgui.Name = "ESP_Text_" .. part.Name
    bgui.Adornee = part
    bgui.Size = UDim2.new(0, 80, 0, 20)
    bgui.AlwaysOnTop = true
    bgui.ExtentsOffset = Vector3.new(0, 1.8, 0)
    bgui.Parent = espFolder

    local txt = Instance.new("TextLabel")
    txt.Size = UDim2.new(1, 0, 1, 0)
    txt.BackgroundTransparency = 0.4
    txt.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
    txt.Text = "Stage " .. part.Name
    txt.TextColor3 = Color3.fromRGB(255, 50, 50)
    txt.Font = Enum.Font.SourceSansBold
    txt.TextSize = 11
    txt.Parent = bgui
    Instance.new("UICorner", txt).CornerRadius = UDim.new(0, 3)

    -- Highlight (Box Stage - LUÔN HIỆN XUYÊN TƯỜNG)
    local hl = Instance.new("Highlight")
    hl.Name = "ESP_HL_" .. part.Name
    hl.Adornee = part
    hl.FillColor = Color3.fromRGB(255, 0, 0)
    hl.FillTransparency = 0.5
    hl.OutlineColor = Color3.fromRGB(255, 255, 255)
    hl.OutlineTransparency = 0
    hl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
    hl.Parent = espFolder

    -- Tracer Line (Dây đỏ nối đến Stage)
    local line = Instance.new("LineHandleAdornment")
    line.Name = "ESP_Line_" .. part.Name
    line.Length = 0
    line.Thickness = 2
    line.Color3 = Color3.fromRGB(255, 0, 0)
    line.AlwaysOnTop = true
    line.ZIndex = 10
    line.Parent = espFolder

    activeESP[part] = {Text = bgui, Label = txt, Highlight = hl, Line = line}
end

local function removeClassicESP(part)
    if activeESP[part] then
        if activeESP[part].Text then activeESP[part].Text:Destroy() end
        if activeESP[part].Highlight then activeESP[part].Highlight:Destroy() end
        if activeESP[part].Line then activeESP[part].Line:Destroy() end
        activeESP[part] = nil
    end
end

RunService.Stepped:Connect(function()
    local hrp = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
    if not hrp or not DEFAULT_SETTINGS.CHECKPOINT_FOLDER then return end

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

    local validTargets = {}

    -- NẾU ĐANG BAY: Giữ nguyên ESP cho Stage đang bay tới + Stage kế tiếp
    if currentTargetPart and currentTargetPart:IsA("BasePart") then
        validTargets[currentTargetPart] = true
        local currentNum = tonumber(currentTargetPart.Name)
        if currentNum then
            local nextPart = DEFAULT_SETTINGS.CHECKPOINT_FOLDER:FindFirstChild(tostring(currentNum + 1))
            if nextPart then validTargets[nextPart] = true end
        end
    else
        -- KHI KHÔNG BAY: Hiện 2 Stage kế tiếp theo khoảng cách
        local target1 = DEFAULT_SETTINGS.CHECKPOINT_FOLDER:FindFirstChild(tostring(closestStageNum + 1))
        local target2 = DEFAULT_SETTINGS.CHECKPOINT_FOLDER:FindFirstChild(tostring(closestStageNum + 2))
        if target1 then validTargets[target1] = true end
        if target2 then validTargets[target2] = true end
    end

    -- Xóa ESP cũ không nằm trong danh sách
    for part in pairs(activeESP) do
        if not validTargets[part] then
            removeClassicESP(part)
        end
    end

    -- Cập nhật ESP & Line
    for part in pairs(validTargets) do
        if not activeESP[part] then
            createClassicESP(part)
        end

        local espData = activeESP[part]
        if espData then
            local dist = math.floor((hrp.Position - part.Position).Magnitude)
            if espData.Label then
                espData.Label.Text = "Stage " .. part.Name .. " [" .. dist .. "m]"
            end

            if espData.Line then
                espData.Line.Adornee = hrp
                espData.Line.CFrame = CFrame.new(Vector3.zero, part.Position - hrp.Position)
                espData.Line.Length = (part.Position - hrp.Position).Magnitude
            end
        end
    end
end)

-- ==========================================
-- MENU LOADING (MADE BY CHIẾN ĐO - 3s)
-- ==========================================
local function createLoadingScreen()
    if not player:FindFirstChild("PlayerGui") then
        repeat task.wait() until player:FindFirstChild("PlayerGui")
    end

    local loadingGui = Instance.new("ScreenGui", player.PlayerGui)
    loadingGui.Name = "Loading_ChienDo"; loadingGui.DisplayOrder = 999
    
    local bg = Instance.new("Frame", loadingGui); bg.Size = UDim2.new(1, 0, 1, 0); bg.BackgroundColor3 = Color3.new(0, 0, 0); bg.BackgroundTransparency = 0.2; bg.BorderSizePixel = 0
    local centerContainer = Instance.new("Frame", bg); centerContainer.Size = UDim2.new(0, 600, 0, 100); centerContainer.Position = UDim2.new(0.5, -300, 0.5, -50); centerContainer.BackgroundTransparency = 1
    local textContainer = Instance.new("Frame", centerContainer); textContainer.Size = UDim2.new(1, 0, 0, 60); textContainer.Position = UDim2.new(0, 0, 0, 0); textContainer.BackgroundTransparency = 1; textContainer.ClipsDescendants = false 
    
    local layout = Instance.new("UIListLayout", textContainer); layout.FillDirection = Enum.FillDirection.Horizontal; layout.HorizontalAlignment = Enum.HorizontalAlignment.Center; layout.VerticalAlignment = Enum.VerticalAlignment.Center; layout.SortOrder = Enum.SortOrder.LayoutOrder

    local subVersionLabel = Instance.new("TextLabel", centerContainer); subVersionLabel.Size = UDim2.new(1, 0, 0, 30); subVersionLabel.Position = UDim2.new(0, 0, 0, 65); subVersionLabel.BackgroundTransparency = 1; subVersionLabel.Text = "MADE BY CHIẾN ĐO"; subVersionLabel.TextColor3 = Color3.fromRGB(180, 180, 180); subVersionLabel.Font = Enum.Font.Gotham; subVersionLabel.TextSize = 16; subVersionLabel.TextTransparency = 1 

    task.spawn(function()
        for i = 1, 120 do
            task.spawn(function()
                local dot = Instance.new("Frame", bg); dot.Size = UDim2.new(0, 5, 0, 5); dot.BackgroundColor3 = Color3.new(1, 0, 0); dot.BorderSizePixel = 0; dot.Position = UDim2.new(math.random(), 0, math.random(), 0); Instance.new("UICorner", dot)
                TweenService:Create(dot, TweenInfo.new(3, Enum.EasingStyle.Linear), {Position = UDim2.new(math.random(), 0, math.random(), 0), BackgroundTransparency = 1}):Play()
            end)
        end
    end)

    local textStr = "script-test"
    local labels = {}
    local order = 1

    for _, c in utf8.codes(textStr) do
        local char = utf8.char(c)
        local charLabel = Instance.new("TextLabel", textContainer)
        charLabel.Size = UDim2.new(0, char == " " and 15 or 30, 1, 0); charLabel.BackgroundTransparency = 1; charLabel.Text = char; charLabel.TextColor3 = Color3.new(1, 0, 0); charLabel.Font = Enum.Font.GothamBold; charLabel.TextSize = 45; charLabel.LayoutOrder = order; charLabel.TextTransparency = 1; charLabel.Position = UDim2.new(0, 0, 0, 60)
        table.insert(labels, charLabel)
        order = order + 1
    end

    task.spawn(function()
        for index, label in ipairs(labels) do
            TweenService:Create(label, TweenInfo.new(0.4, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Position = UDim2.new(0, 0, 0, 0), TextTransparency = 0}):Play()
            task.wait(0.1) 
        end
        TweenService:Create(subVersionLabel, TweenInfo.new(0.5, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {TextTransparency = 0}):Play()
    end)

    task.wait(3)
    loadingGui:Destroy()
end

-- ==========================================
-- HÀM KÉO THẢ UI
-- ==========================================
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

-- ==========================================
-- GIAO DIỆN CHÍNH
-- ==========================================
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
local title = Instance.new("TextLabel", topContent); title.Size = UDim2.new(1, 0, 0, 30); title.Position = UDim2.new(0, 0, 0, 15); title.Text = "script-test"; title.TextColor3 = Color3.new(1, 1, 1); title.Font = Enum.Font.GothamBold; title.TextSize = 18; title.BackgroundTransparency = 1; title.ZIndex = 6
local subTitle = Instance.new("TextLabel", topContent); subTitle.Size = UDim2.new(1, 0, 0, 20); subTitle.Position = UDim2.new(0, 0, 0, 40); subTitle.Text = "(AFK OR PLAY)"; subTitle.TextColor3 = Color3.fromRGB(200, 200, 200); subTitle.Font = Enum.Font.Gotham; subTitle.TextSize = 14; subTitle.BackgroundTransparency = 1; subTitle.ZIndex = 6

local infoBtn = Instance.new("TextButton", frame); infoBtn.Size = UDim2.new(0, 25, 0, 25); infoBtn.Position = UDim2.new(0, 5, 0, 5); infoBtn.Text = "!"; infoBtn.TextColor3 = Color3.new(1, 0, 0); infoBtn.BackgroundTransparency = 1; infoBtn.Font = Enum.Font.GothamBold; infoBtn.TextSize = 20; infoBtn.ZIndex = 10

local bugReportFrame = Instance.new("TextButton", mainGui); bugReportFrame.Size = UDim2.new(0, 320, 0, 180); bugReportFrame.Position = UDim2.new(0.5, -160, 0.5, -90); bugReportFrame.BackgroundColor3 = Color3.fromRGB(10, 10, 10); bugReportFrame.BackgroundTransparency = 0.25; bugReportFrame.Visible = false; bugReportFrame.ZIndex = 100
Instance.new("UICorner", bugReportFrame).CornerRadius = UDim.new(0, 8)

local stroke = Instance.new("UIStroke", bugReportFrame); stroke.Color = Color3.new(1, 0, 0); stroke.Thickness = 2; stroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border

local bugTextLabel = Instance.new("TextLabel", bugReportFrame); bugTextLabel.Size = UDim2.new(1, -20, 1, -20); bugTextLabel.Position = UDim2.new(0, 10, 0, 10); bugTextLabel.BackgroundTransparency = 1; bugTextLabel.Text = "-INFO BOARD-\n-Update 1.101😎\n+fixed bug at stage 243\n+fixed anti-cheat kick (2-3s random pause)\n-Next Update 1.2👍\n+add automatic rebirth\n+control panel"; bugTextLabel.TextColor3 = Color3.new(1, 1, 1); bugTextLabel.Font = Enum.Font.GothamMedium; bugTextLabel.TextSize = 14; bugTextLabel.TextWrapped = true; bugTextLabel.TextXAlignment = Enum.TextXAlignment.Left; bugTextLabel.TextYAlignment = Enum.TextYAlignment.Top; bugTextLabel.ZIndex = 101

infoBtn.MouseButton1Click:Connect(function() bugReportFrame.Visible = not bugReportFrame.Visible end)
bugReportFrame.MouseButton1Click:Connect(function() bugReportFrame.Visible = false end)

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

-- ==========================================
-- BẢO VỆ CHỐNG CHẾT & RUNG CAMERA
-- ==========================================
local function toggleProtection(state)
    if state then
        if not protectionConnection then
            protectionConnection = RunService.Stepped:Connect(function()
                if running and player.Character then
                    local h = player.Character:FindFirstChildOfClass("Humanoid")
                    if h then 
                        h.MaxHealth = math.huge
                        h.Health = math.huge
                        h:SetStateEnabled(Enum.HumanoidStateType.Dead, false)
                    end
                    for _, p in pairs(player.Character:GetDescendants()) do 
                        if p:IsA("BasePart") then 
                            p.CanCollide = false 
                            p.AssemblyLinearVelocity = Vector3.zero
                            p.AssemblyAngularVelocity = Vector3.zero
                        end 
                    end
                end
            end)
        end
    else
        if protectionConnection then protectionConnection:Disconnect(); protectionConnection = nil end
    end
end

-- ==========================================
-- LOGIC DI CHUYỂN CHÍNH
-- ==========================================
local function flyToTarget(target)
    currentTargetPart = target
    local hrp = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
    if not hrp or not target or not running then 
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
        if not running or not hrp or not hrp.Parent then
            if moveConnection then moveConnection:Disconnect() end
            return
        end
        
        local elapsed = tick() - startTime
        local alpha = math.clamp(elapsed / duration, 0, 1)
        
        hrp.CFrame = startCFrame:Lerp(targetCFrame, alpha)
        
        if alpha >= 1 then
            if moveConnection then moveConnection:Disconnect() end
        end
    end)
    
    while running and (tick() - startTime) < duration do
        task.wait()
    end
    
    if moveConnection then moveConnection:Disconnect() end
    if not running then 
        currentTargetPart = nil
        return 
    end

    hrp.CFrame = target.CFrame * CFrame.new(0, 1.5, 0)
    
    if firetouchinterest and target:IsA("BasePart") then
        firetouchinterest(hrp, target, 0)
        task.wait(0.05)
        firetouchinterest(hrp, target, 1)
    end
    
    -- ĐOẠN XỬ LÝ STAGE 243
    if target.Name == "243" and running then
        task.wait(0.28)
        if running and player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
            local currentHrp = player.Character.HumanoidRootPart
            local startPos = currentHrp.Position
            local blueDirectionCFrame = CFrame.new(startPos.X, startPos.Y, startPos.Z + 200)
            
            local duration243 = 200 / currentFlySpeed
            local start243 = tick()
            
            local conn243
            conn243 = RunService.RenderStepped:Connect(function()
                if not running or not currentHrp then if conn243 then conn243:Disconnect() end return end
                local elapsed = tick() - start243
                local alpha = math.clamp(elapsed / duration243, 0, 1)
                currentHrp.CFrame = CFrame.new(startPos):Lerp(blueDirectionCFrame, alpha)
                if alpha >= 1 then if conn243 then conn243:Disconnect() end end
            end)
            
            while running and (tick() - start243) < duration243 do task.wait() end
            if conn243 then conn243:Disconnect() end
        end
    end
    
    toggleProtection(false)
    local pauseTime
    if target.Name == "243" then
        pauseTime = math.random(20, 30) / 10
    else
        pauseTime = math.random(3, 5) / 10
    end
    task.wait(pauseTime)
    
    currentTargetPart = nil
    
    if running then
        toggleProtection(true)
    end
end

local function startFarming()
    running = true
    wasRunningBeforeDeath = true
    toggleProtection(true)
    btn.Text = "Auto Farm Stage: ON"; btn.BackgroundColor3 = Color3.new(1, 0, 0); btn.TextColor3 = Color3.new(1, 1, 1)
    
    task.spawn(function()
        local lastStageNum = -1
        local stuckTime = 0
        local isSkippingStuck = false
        
        while running do
            local hrp = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
            if not hrp then task.wait(1) continue end
            
            local closestStageNum = -1
            local minDist = math.huge
            
            if DEFAULT_SETTINGS.CHECKPOINT_FOLDER then
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
            end
            
            if closestStageNum == lastStageNum then
                stuckTime = stuckTime + 0.5
                if stuckTime >= 8 and not isSkippingStuck then
                    isSkippingStuck = true
                    local nextStageObj = DEFAULT_SETTINGS.CHECKPOINT_FOLDER and DEFAULT_SETTINGS.CHECKPOINT_FOLDER:FindFirstChild(tostring(closestStageNum + 1))
                    
                    if nextStageObj then
                        flyToTarget(nextStageObj)
                    else
                        autoRejoin()
                    end
                    
                    stuckTime = 0
                    isSkippingStuck = false
                    task.wait(0.5)
                    continue
                end
            else
                lastStageNum = closestStageNum
                stuckTime = 0
            end
            
            local nextTarget = DEFAULT_SETTINGS.CHECKPOINT_FOLDER and DEFAULT_SETTINGS.CHECKPOINT_FOLDER:FindFirstChild(tostring(closestStageNum + 1))
            
            if nextTarget then
                flyToTarget(nextTarget)
            else
                task.wait(0.5)
            end
        end
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

-- ==========================================
-- XỬ LÝ CHẾT / RESPAWN
-- ==========================================
player.CharacterAdded:Connect(function(newChar) 
    toggleProtection(false)
    currentTargetPart = nil
    
    if wasRunningBeforeDeath then
        task.wait(1)
        if wasRunningBeforeDeath then
            startFarming()
        end
    else
        btn.Text = "Auto Farm Stage: OFF"; btn.BackgroundColor3 = Color3.new(1, 1, 1); btn.TextColor3 = Color3.new(0, 0, 0)
    end
end)

if _G.IsAutoRejoin then
    _G.IsAutoRejoin = nil
    task.spawn(function()
        task.wait(3.5)
        if not running then
            startFarming()
        end
    end)
end
