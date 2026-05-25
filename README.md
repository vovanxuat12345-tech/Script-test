local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local TeleportService = game:GetService("TeleportService")
local GuiService = game:GetService("GuiService")
local player = Players.LocalPlayer

-- ========================================================
-- KIỂM TRA CHỐNG TRÙNG SCRIPT TOÀN CỤC (SỬA LỖI REJOIN TRÙNG)
-- ========================================================
if _G.ScriptRunning then
    return
end
_G.ScriptRunning = true 

-- Tự xóa UI cũ nếu còn sót lại từ server trước
local oldGui = player:WaitForChild("PlayerGui"):FindFirstChild("AutoObby_GodAlways")
if oldGui then oldGui:Destroy() end
local oldLoading = player:WaitForChild("PlayerGui"):FindFirstChild("Loading_ChienDo")
if oldLoading then oldLoading:Destroy() end

-- ==========================================
-- ĐOẠN XỬ LÝ AUTO REJOIN CHUẨN (0.5S)
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
            
            loadstring(game:HttpGet("https://raw.githubusercontent.com/vovanxuat12345-tech/Script-test/refs/heads/main/README.md"))()
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

-- ==========================================
-- MENU LOADING CHỮ MÀU ĐỎ + SUBTITLE ROBTOP
-- ==========================================
local function createLoadingScreen()
    if not player:FindFirstChild("PlayerGui") then
        repeat task.wait() until player:FindFirstChild("PlayerGui")
    end

    local loadingGui = Instance.new("ScreenGui", player.PlayerGui)
    loadingGui.Name = "Loading_ChienDo"; loadingGui.DisplayOrder = 999
    
    local bg = Instance.new("Frame", loadingGui)
    bg.Size = UDim2.new(1, 0, 1, 0)
    bg.BackgroundColor3 = Color3.new(0, 0, 0)
    bg.BackgroundTransparency = 0.2
    bg.BorderSizePixel = 0
    
    -- Khung tổng chứa cả tên chính và sub nhỏ để căn giữa màn hình dễ hơn
    local centerContainer = Instance.new("Frame", bg)
    centerContainer.Size = UDim2.new(0, 600, 0, 100)
    centerContainer.Position = UDim2.new(0.5, -300, 0.5, -50)
    centerContainer.BackgroundTransparency = 1
    
    -- Khung chứa văn bản chính (MADE BY CHIẾN ĐO)
    local textContainer = Instance.new("Frame", centerContainer)
    textContainer.Size = UDim2.new(1, 0, 0, 60)
    textContainer.Position = UDim2.new(0, 0, 0, 0)
    textContainer.BackgroundTransparency = 1
    textContainer.ClipsDescendants = false 
    
    -- Tự động sắp xếp các chữ cái từ trái sang phải
    local layout = Instance.new("UIListLayout", textContainer)
    layout.FillDirection = Enum.FillDirection.Horizontal
    layout.HorizontalAlignment = Enum.HorizontalAlignment.Center
    layout.VerticalAlignment = Enum.VerticalAlignment.Center
    layout.SortOrder = Enum.SortOrder.LayoutOrder

    -- Chữ nhỏ ở dưới theo yêu cầu bồ nè
    local subVersionLabel = Instance.new("TextLabel", centerContainer)
    subVersionLabel.Size = UDim2.new(1, 0, 0, 30)
    subVersionLabel.Position = UDim2.new(0, 0, 0, 65) 
    subVersionLabel.BackgroundTransparency = 1
    subVersionLabel.Text = "(version 2.209 By RobTop)"
    subVersionLabel.TextColor3 = Color3.fromRGB(180, 180, 180) 
    subVersionLabel.Font = Enum.Font.Gotham
    subVersionLabel.TextSize = 16
    subVersionLabel.TextTransparency = 1 

    -- Tạo các chấm đỏ chuyển động nền
    task.spawn(function()
        for i = 1, 120 do
            task.spawn(function()
                local dot = Instance.new("Frame", bg); dot.Size = UDim2.new(0, 5, 0, 5); dot.BackgroundColor3 = Color3.new(1, 0, 0); dot.BorderSizePixel = 0; dot.Position = UDim2.new(math.random(), 0, math.random(), 0); Instance.new("UICorner", dot)
                TweenService:Create(dot, TweenInfo.new(3, Enum.EasingStyle.Linear), {Position = UDim2.new(math.random(), 0, math.random(), 0), BackgroundTransparency = 1}):Play()
            end)
        end
    end)

    -- Chuỗi chữ cần chạy hiệu ứng
    local textStr = "MADE BY CHIẾN ĐO"
    local labels = {}
    local order = 1

    -- Sử dụng thư viện utf8 để cắt chính xác chữ tiếng Việt có dấu
    for _, c in utf8.codes(textStr) do
        local char = utf8.char(c)
        local charLabel = Instance.new("TextLabel", textContainer)
        
        charLabel.Size = UDim2.new(0, char == " " and 15 or 30, 1, 0)
        charLabel.BackgroundTransparency = 1
        charLabel.Text = char
        charLabel.TextColor3 = Color3.new(1, 0, 0) 
        charLabel.Font = Enum.Font.GothamBold
        charLabel.TextSize = 45
        charLabel.LayoutOrder = order
        
        -- Vị trí ban đầu: Ẩn hoàn toàn và tụt xuống dưới 60 pixel
        charLabel.TextTransparency = 1
        charLabel.Position = UDim2.new(0, 0, 0, 60) 
        
        table.insert(labels, charLabel)
        order = order + 1
    end

    -- Thực hiện Animation chạy từng chữ từ trái sang phải
    task.spawn(function()
        for index, label in ipairs(labels) do
            TweenService:Create(label, TweenInfo.new(0.4, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
                Position = UDim2.new(0, 0, 0, 0),
                TextTransparency = 0
            }):Play()
            
            task.wait(0.1) 
        end
        -- Sau khi chữ chính chạy gần xong, cho hiện chữ phiên bản RobTop lên mượt mà
        TweenService:Create(subVersionLabel, TweenInfo.new(0.5, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
            TextTransparency = 0
        }):Play()
    end)

    task.wait(3) -- Giữ nguyên màn hình đúng 3 giây
    loadingGui:Destroy()
end
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

-- Nút thông báo dấu chấm than "!" ở góc trên cùng bên trái
local infoBtn = Instance.new("TextButton", frame)
infoBtn.Size = UDim2.new(0, 25, 0, 25)
infoBtn.Position = UDim2.new(0, 5, 0, 5)
infoBtn.Text = "!"
infoBtn.TextColor3 = Color3.new(1, 0, 0) -- Màu đỏ cảnh báo
infoBtn.BackgroundTransparency = 1
infoBtn.Font = Enum.Font.GothamBold
infoBtn.TextSize = 20
infoBtn.ZIndex = 10

-- Bảng thông báo lỗi (Đen mờ, viền đỏ, nằm giữa màn hình)
local bugReportFrame = Instance.new("TextButton", mainGui) -- Dùng TextButton để người chơi bấm vào tự tắt nhanh được luôn
bugReportFrame.Size = UDim2.new(0, 320, 0, 140)
bugReportFrame.Position = UDim2.new(0.5, -160, 0.5, -70)
bugReportFrame.BackgroundColor3 = Color3.fromRGB(10, 10, 10)
bugReportFrame.BackgroundTransparency = 0.25 -- Đen mờ theo yêu cầu
bugReportFrame.Visible = false
bugReportFrame.ZIndex = 100
Instance.new("UICorner", bugReportFrame).CornerRadius = UDim.new(0, 8)

-- Tạo viền màu đỏ cho bảng thông báo
local stroke = Instance.new("UIStroke", bugReportFrame)
stroke.Color = Color3.new(1, 0, 0) -- Viền đỏ
stroke.Thickness = 2
stroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border

-- Nội dung chữ tiếng Anh bên trong bảng thông báo
local bugTextLabel = Instance.new("TextLabel", bugReportFrame)
bugTextLabel.Size = UDim2.new(1, -20, 1, -20)
bugTextLabel.Position = UDim2.new(0, 10, 0, 10)
bugTextLabel.BackgroundTransparency = 1
bugTextLabel.Text = "I'm sorry, but there are still a few bugs that I haven't been able to fix yet, such as at stage 243. After all, I'm truly sorry everyone."
bugTextLabel.TextColor3 = Color3.new(1, 1, 1)
bugTextLabel.Font = Enum.Font.GothamMedium
bugTextLabel.TextSize = 14
bugTextLabel.TextWrapped = true
bugTextLabel.ZIndex = 101

-- Logic bật tắt bảng thông báo lỗi
infoBtn.MouseButton1Click:Connect(function()
    bugReportFrame.Visible = not bugReportFrame.Visible
end)

bugReportFrame.MouseButton1Click:Connect(function()
    bugReportFrame.Visible = false
end)

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
    
    -- ========================================================
    -- ĐÃ SỬA: PHÁT HIỆN CHECKPOINT "243" THÌ BAY SANG TRÁI 1000 STUDS
    -- ========================================================
    if target.Name == "243" and running then
        -- Lấy Vector hướng bên trái nhân vật dựa vào CFrame hiện tại (-RightVector)
        local leftDirection = -hrp.CFrame.RightVector
        local leftTargetPos = hrp.Position + (leftDirection * 1000)
        
        -- Tiến hành dùng Vận tốc (Fly) bằng biến currentFlySpeed để bay sang trái
        while running and (Vector2.new(hrp.Position.X, hrp.Position.Z) - Vector2.new(leftTargetPos.X, leftTargetPos.Z)).Magnitude > 5 do
            hrp.Velocity = leftDirection * currentFlySpeed
            task.wait()
        end
        hrp.Velocity = Vector3.new(0, 0.2, 0) -- Reset lại vận tốc sau khi bay xong
    end
    -- ========================================================
    
    currentFlySpeed = DEFAULT_SETTINGS.FLY_SPEED
    scanMultiplier = 1
    
    task.wait(DEFAULT_SETTINGS.WAIT_TIME) 
end

local function startFarming()
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

local function stopFarming()
    running = false; toggleProtection(false)
    btn.Text = "Auto Farm Stage: OFF"; btn.BackgroundColor3 = Color3.new(1, 1, 1); btn.TextColor3 = Color3.new(0, 0, 0)
end

btn.MouseButton1Click:Connect(function()
    if running then stopFarming() else startFarming() end
end)

player.CharacterAdded:Connect(function() 
    running = false; toggleProtection(false)
    currentFlySpeed = DEFAULT_SETTINGS.FLY_SPEED
    scanMultiplier = 1
    btn.Text = "Auto Farm Stage: OFF"; btn.BackgroundColor3 = Color3.new(1, 1, 1) 
end)

-- KÍCH HOẠT AUTO FARM CHỈ KHI REJOIN THÀNH CÔNG
if _G.IsAutoRejoin then
    _G.IsAutoRejoin = nil
    task.spawn(function()
        task.wait(3.5) -- Đợi hiệu ứng chữ chạy xong hoàn toàn (3.5s) rồi bắt đầu farm
        if not running then
            startFarming()
        end
    end)
end
