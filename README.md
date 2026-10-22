local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local TeleportService = game:GetService("TeleportService")
local GuiService = game:GetService("GuiService")
local StarterGui = game:GetService("StarterGui")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local VirtualUser = game:GetService("VirtualUser")
local Workspace = game:GetService("Workspace")

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
        if #Players:GetPlayers() <= 1 then TeleportService:Teleport(TARGET_PLACE_ID, player)
        else TeleportService:TeleportToPlaceInstance(TARGET_PLACE_ID, game.JobId, player) end
    end)
    task.delay(12, function()
        if isRejoining and game.PlaceId == TARGET_PLACE_ID then TeleportService:Teleport(TARGET_PLACE_ID, player) end
    end)
end

GuiService.ErrorMessageChanged:Connect(function() if game.PlaceId == TARGET_PLACE_ID then autoRejoin() end end)
game:GetService("CoreGui").RobloxPromptGui.promptOverlay.ChildAdded:Connect(function(c) if c.Name == "ErrorPrompt" and game.PlaceId == TARGET_PLACE_ID then autoRejoin() end end)
TeleportService.TeleportInitFailed:Connect(function() if game.PlaceId == TARGET_PLACE_ID then isRejoining = false; task.wait(1); autoRejoin() end end)

local shaderEnabled = true
local SKY_NAME = "SoftMorning_ShaderSky"

local lightingService = game:GetService("Lighting")
local origLighting = {
    Tech = pcall(function() return lightingService.Technology end) and lightingService.Technology or Enum.Technology.Compatibility,
    Shadows = lightingService.GlobalShadows,
    Clock = lightingService.ClockTime,
    Bright = lightingService.Brightness,
    Exp = lightingService.ExposureCompensation,
    Amb = lightingService.Ambient,
    OutAmb = lightingService.OutdoorAmbient,
    Top = lightingService.ColorShift_Top
}

local function disableShader()
    local ls = game:GetService("Lighting")
    for _, obj in ipairs(ls:GetChildren()) do
        if obj.Name == SKY_NAME or obj:GetAttribute("IsShaderEffect") == true then
            pcall(function() obj:Destroy() end)
        end
    end
    pcall(function()
        ls.Technology = origLighting.Tech
        ls.GlobalShadows = origLighting.Shadows
        ls.ClockTime = origLighting.Clock
        ls.Brightness = origLighting.Bright
        ls.ExposureCompensation = origLighting.Exp
        ls.Ambient = origLighting.Amb
        ls.OutdoorAmbient = origLighting.OutAmb
        ls.ColorShift_Top = origLighting.Top
    end)
end

local function enableShader()
    disableShader()
    local ls = game:GetService("Lighting")
    
    local successSky, skyErr = pcall(function()
        local sky = Instance.new("Sky")
        sky.Name = SKY_NAME
        sky.SkyboxBk = "rbxassetid://153723321"
        sky.SkyboxDn = "rbxassetid://153723315"
        sky.SkyboxFt = "rbxassetid://153723330"
        sky.SkyboxLf = "rbxassetid://153723337"
        sky.SkyboxRt = "rbxassetid://153723348"
        sky.SkyboxUp = "rbxassetid://153723354"
        sky.SunTextureId = "rbxassetid://10842211244"
        sky.SunAngularSize = 15
        sky.CelestialBodiesShown = true
        sky.Parent = ls
    end)
    if not successSky then warn("[Shader Error] Sky:", skyErr) end

    local function createEffect(className, name, properties)
        pcall(function()
            local effect = Instance.new(className)
            effect.Name = name
            effect:SetAttribute("IsShaderEffect", true)
            for prop, val in pairs(properties) do
                effect[prop] = val
            end
            effect.Parent = ls
        end)
    end

    pcall(function() ls.Technology = Enum.Technology.Future end)
    ls.GlobalShadows = true
    ls.ClockTime = 10.0
    ls.Brightness = 1.2
    ls.ExposureCompensation = 0.0
    ls.Ambient = Color3.fromRGB(110, 110, 120)
    ls.OutdoorAmbient = Color3.fromRGB(130, 135, 140)
    ls.ColorShift_Top = Color3.fromRGB(255, 255, 255)

    createEffect("Atmosphere", "Shader_Atmosphere", {
        Density = 0.15, 
        Offset = 0.2,
        Color = Color3.fromRGB(180, 210, 245), 
        Decay = Color3.fromRGB(140, 160, 180), 
        Haze = 0.5
    })
    
    createEffect("SunRaysEffect", "Shader_SunRays", {Intensity = 0.15, Spread = 0.5})
    createEffect("BloomEffect", "Shader_Bloom", {Intensity = 0.1, Size = 12, Threshold = 0.95})
    
    createEffect("ColorCorrectionEffect", "Shader_ColorCorrection", {
        Brightness = 0.0, 
        Contrast = 0.1, 
        Saturation = 0.15, 
        TintColor = Color3.fromRGB(255, 255, 255)
    })
end

local function FixBlockMaterial(part)
    if part:IsA("BasePart") then
        local isP = false
        pcall(function() if player.Character and part:IsDescendantOf(player.Character) then isP = true end end)
        if not isP and part.Transparency < 0.2 then
            part.Reflectance = 0.15
        end
    end
end

task.spawn(function() pcall(function() for _, obj in ipairs(Workspace:GetDescendants()) do FixBlockMaterial(obj) end end) end)
Workspace.DescendantAdded:Connect(function(child) task.wait(0.15); FixBlockMaterial(child) end)
if shaderEnabled then enableShader() end

local DEFAULT_SETTINGS = {FLY_SPEED = 40, CHECKPOINT_FOLDER = Workspace:WaitForChild("Checkpoints", 15)}
local currentFlySpeed = DEFAULT_SETTINGS.FLY_SPEED
local running, wasRunningBeforeDeath, isMinimized, isSettingsOpen = false, false, false, false
local espEnabled, playerEspEnabled, playerHighlightEnabled, afkModeEnabled = true, false, false, false
local autoRebirthEnabled, antiAfkEnabled, freeCamEnabled = true, true, false
local stage251ReachedTime, lastJumpTime, protectionConnection, currentTargetPart, farmingTask = 0, tick(), nil, nil, nil
local freeCamPos, freeCamYaw, freeCamPitch, freeCamSpeed, freeCamSens = Vector3.new(), 0, 0, 50, 0.0125
local savedWalkSpeed, savedJumpPower = 16, 50
local afkStartTime, afkStartStage, stagesGained, lastNotifiedMilestone = 0, 0, 0, -1

local isDraggingFly = false
local isDraggingFc = false

local function sendNotification(title, text) pcall(function() StarterGui:SetCore("SendNotification", {Title = title, Text = text, Duration = 4}) end) end
local function triggerRebirth() pcall(function() local r = ReplicatedStorage:FindFirstChild("MainRemote"); if r then r:FireServer({Request = "RequestRebirth"}) end end) end

local function lockPlayerCharacter(lock)
    pcall(function()
        local char = player.Character
        local hum = char and char:FindFirstChildOfClass("Humanoid")
        if hum then
            if lock then
                savedWalkSpeed, savedJumpPower = hum.WalkSpeed > 0 and hum.WalkSpeed or 16, hum.JumpPower > 0 and hum.JumpPower or 50
                hum.WalkSpeed, hum.JumpPower, hum.JumpHeight = 0, 0, 0
            else
                hum.WalkSpeed, hum.JumpPower, hum.JumpHeight = savedWalkSpeed, savedJumpPower, savedJumpPower
            end
        end
    end)
end

local espGui = game:GetService("CoreGui"):FindFirstChild("Classic_Stage_ESP") or Instance.new("ScreenGui", game:GetService("CoreGui"))
espGui.Name = "Classic_Stage_ESP"; espGui:ClearAllChildren()
local activeESP, activePlayerESP = {}, {}

local function createClassicESP(part)
    if not part or not part:IsA("BasePart") or activeESP[part] then return end
    local bgui = Instance.new("BillboardGui", espGui); bgui.Name, bgui.Adornee, bgui.Size, bgui.AlwaysOnTop, bgui.ExtentsOffset = "ESP_Text_" .. part.Name, part, UDim2.new(0, 80, 0, 20), true, Vector3.new(0, 1.8, 0)
    local txt = Instance.new("TextLabel", bgui); txt.Size, txt.BackgroundTransparency, txt.BackgroundColor3, txt.Text, txt.TextColor3, txt.Font, txt.TextSize = UDim2.new(1,0,1,0), 0.4, Color3.fromRGB(0,0,0), "Stage " .. part.Name, Color3.fromRGB(255,50,50), Enum.Font.SourceSansBold, 11
    Instance.new("UICorner", txt).CornerRadius = UDim.new(0, 3)
    local hl = Instance.new("Highlight", espGui); hl.Name, hl.Adornee, hl.FillColor, hl.FillTransparency, hl.OutlineColor, hl.OutlineTransparency, hl.DepthMode = "ESP_HL_" .. part.Name, part, Color3.fromRGB(255,0,0), 0.5, Color3.fromRGB(255,255,255), 0, Enum.HighlightDepthMode.AlwaysOnTop
    local line2d = Drawing.new("Line"); line2d.Color, line2d.Thickness, line2d.Transparency, line2d.Visible = Color3.fromRGB(255,0,0), 0.5, 1, false
    activeESP[part] = {Text = bgui, Label = txt, Highlight = hl, Line2D = line2d}
end

local function removeClassicESP(part)
    if activeESP[part] then
        if activeESP[part].Text then activeESP[part].Text:Destroy() end
        if activeESP[part].Highlight then activeESP[part].Highlight:Destroy() end
        if activeESP[part].Line2D then pcall(function() activeESP[part].Line2D:Remove() end) end
        activeESP[part] = nil
    end
end

local function createPlayerESP(otherPlayer)
    if not otherPlayer or otherPlayer == player or activePlayerESP[otherPlayer] then return end
    local line2d = Drawing.new("Line"); line2d.Color, line2d.Thickness, line2d.Transparency, line2d.Visible = Color3.fromRGB(0,150,255), 0.5, 1, false
    local bgui = Instance.new("BillboardGui", espGui); bgui.Name, bgui.Size, bgui.AlwaysOnTop, bgui.ExtentsOffset = "P_ESP_" .. otherPlayer.Name, UDim2.new(0,100,0,20), true, Vector3.new(0,2.5,0)
    local txt = Instance.new("TextLabel", bgui); txt.Size, txt.BackgroundTransparency, txt.BackgroundColor3, txt.Text, txt.TextColor3, txt.Font, txt.TextSize = UDim2.new(1,0,1,0), 0.5, Color3.fromRGB(0,0,0), otherPlayer.DisplayName, Color3.fromRGB(0,180,255), Enum.Font.SourceSansBold, 11
    Instance.new("UICorner", txt).CornerRadius = UDim.new(0, 3)
    activePlayerESP[otherPlayer] = {Line2D = line2d, Billboard = bgui, TextLabel = txt}
end

local function removePlayerESP(otherPlayer)
    if activePlayerESP[otherPlayer] then
        if activePlayerESP[otherPlayer].Line2D then pcall(function() activePlayerESP[otherPlayer].Line2D:Remove() end) end
        if activePlayerESP[otherPlayer].Billboard then activePlayerESP[otherPlayer].Billboard:Destroy() end
        activePlayerESP[otherPlayer] = nil
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
        playerHLObj = Instance.new("Highlight", espGui); playerHLObj.Name, playerHLObj.Adornee, playerHLObj.FillTransparency, playerHLObj.OutlineColor, playerHLObj.OutlineTransparency, playerHLObj.DepthMode = "Player_Self_Outline", char, 1, Color3.fromRGB(255,0,0), 0, Enum.HighlightDepthMode.AlwaysOnTop
    end
end

local isRightDragging = false
UserInputService.InputBegan:Connect(function(input) 
    if freeCamEnabled and not isDraggingFly and not isDraggingFc and input.UserInputType == Enum.UserInputType.MouseButton2 then 
        isRightDragging = true 
    end 
end)
UserInputService.InputEnded:Connect(function(input) 
    if input.UserInputType == Enum.UserInputType.MouseButton2 then 
        isRightDragging = false 
    end 
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        isDraggingFly = false
        isDraggingFc = false
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if not freeCamEnabled or isDraggingFly or isDraggingFc then return end
    if input.UserInputType == Enum.UserInputType.MouseMovement and isRightDragging then
        local delta = input.Delta; freeCamYaw = freeCamYaw - (delta.X * freeCamSens); freeCamPitch = math.clamp(freeCamPitch - (delta.Y * freeCamSens), math.rad(-80), math.rad(80))
    elseif input.UserInputType == Enum.UserInputType.Touch and input.Position.X >= (camera.ViewportSize.X / 2) then
        local delta = input.Delta; freeCamYaw = freeCamYaw - (delta.X * freeCamSens); freeCamPitch = math.clamp(freeCamPitch - (delta.Y * freeCamSens), math.rad(-80), math.rad(80))
    end
end)

RunService.RenderStepped:Connect(function(dt)
    updatePlayerHighlight()
    local lsRun = game:GetService("Lighting")
    if shaderEnabled and math.abs(lsRun.ClockTime - 10.0) > 0.05 then lsRun.ClockTime = 10.0; lsRun.Brightness = 1.2 end

    if freeCamEnabled then
        camera.CameraType = Enum.CameraType.Scriptable
        local camRot = CFrame.Angles(0, freeCamYaw, 0) * CFrame.Angles(freeCamPitch, 0, 0)
        local moveVector = Vector3.new()
        pcall(function() moveVector = require(player.PlayerScripts:WaitForChild("PlayerModule")):GetControls():GetMoveVector() end)
        if moveVector.Magnitude > 0 then freeCamPos = freeCamPos + (camRot:VectorToWorldSpace(Vector3.new(moveVector.X, 0, moveVector.Z)) * freeCamSpeed * dt) end
        pcall(function() local hum = player.Character and player.Character:FindFirstChildOfClass("Humanoid"); if hum and hum.WalkSpeed ~= 0 then hum.WalkSpeed = 0 end end)
        camera.CFrame = CFrame.new(freeCamPos) * camRot
    else
        if camera.CameraType == Enum.CameraType.Scriptable and not afkModeEnabled then camera.CameraType = Enum.CameraType.Custom end
    end

    local hrp = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
    if not espEnabled or afkModeEnabled then
        for part in pairs(activeESP) do removeClassicESP(part) end
    elseif hrp and DEFAULT_SETTINGS.CHECKPOINT_FOLDER then
        local closestStageNum, minDist = -1, math.huge
        for _, cp in pairs(DEFAULT_SETTINGS.CHECKPOINT_FOLDER:GetChildren()) do
            local stageNum = tonumber(cp.Name)
            if stageNum and cp:IsA("BasePart") then local d = (hrp.Position - cp.Position).Magnitude; if d < minDist then minDist = d; closestStageNum = stageNum end end
        end
        local validTargets = {}
        if currentTargetPart and currentTargetPart:IsA("BasePart") then
            validTargets[currentTargetPart] = true
            local num = tonumber(currentTargetPart.Name); local nextPart = num and DEFAULT_SETTINGS.CHECKPOINT_FOLDER:FindFirstChild(tostring(num + 1))
            if nextPart then validTargets[nextPart] = true end
        else
            local t1, t2 = DEFAULT_SETTINGS.CHECKPOINT_FOLDER:FindFirstChild(tostring(closestStageNum + 1)), DEFAULT_SETTINGS.CHECKPOINT_FOLDER:FindFirstChild(tostring(closestStageNum + 2))
            if t1 then validTargets[t1] = true end; if t2 then validTargets[t2] = true end
        end
        for part in pairs(activeESP) do if not validTargets[part] then removeClassicESP(part) end end
        for part in pairs(validTargets) do
            if not activeESP[part] then createClassicESP(part) end
            local data = activeESP[part]
            if data then
                local basePos = freeCamEnabled and freeCamPos or hrp.Position
                if data.Label then data.Label.Text = "Stage " .. part.Name .. " [" .. math.floor((basePos - part.Position).Magnitude) .. "m]" end
                if data.Line2D then
                    local fromV = freeCamEnabled and Vector2.new(camera.ViewportSize.X/2, camera.ViewportSize.Y/2) or Vector2.new(camera:WorldToViewportPoint((player.Character:FindFirstChild("Head") or hrp).Position).X, camera:WorldToViewportPoint((player.Character:FindFirstChild("Head") or hrp).Position).Y)
                    local p2 = camera:WorldToViewportPoint(part.Position); local toV = Vector2.new(p2.X, p2.Y)
                    if p2.Z < 0 then toV = Vector2.new(camera.ViewportSize.X/2, camera.ViewportSize.Y/2) - (toV - Vector2.new(camera.ViewportSize.X/2, camera.ViewportSize.Y/2)).Unit * 1000 end
                    data.Line2D.From, data.Line2D.To, data.Line2D.Visible = fromV, toV, true
                end
            end
        end
    end

    if not playerEspEnabled or afkModeEnabled then
        for p in pairs(activePlayerESP) do removePlayerESP(p) end
    else
        for _, op in pairs(Players:GetPlayers()) do
            if op ~= player then
                local oChar = op.Character; local oHrp = oChar and oChar:FindFirstChild("HumanoidRootPart")
                if oHrp then
                    if not activePlayerESP[op] then createPlayerESP(op) end
                    local pData = activePlayerESP[op]
                    if pData then
                        if pData.Billboard then pData.Billboard.Adornee = oHrp end
                        local basePos = freeCamEnabled and freeCamPos or (hrp and hrp.Position or Vector3.new())
                        if pData.TextLabel then pData.TextLabel.Text = op.DisplayName .. " [" .. math.floor((basePos - oHrp.Position).Magnitude) .. "m]" end
                        if pData.Line2D then
                            local fromV = freeCamEnabled and Vector2.new(camera.ViewportSize.X/2, camera.ViewportSize.Y/2) or Vector2.new(camera:WorldToViewportPoint((player.Character:FindFirstChild("Head") or hrp).Position).X, camera:WorldToViewportPoint((player.Character:FindFirstChild("Head") or hrp).Position).Y)
                            local p2 = camera:WorldToViewportPoint(oHrp.Position); local toV = Vector2.new(p2.X, p2.Y)
                            if p2.Z < 0 then toV = Vector2.new(camera.ViewportSize.X/2, camera.ViewportSize.Y/2) - (toV - Vector2.new(camera.ViewportSize.X/2, camera.ViewportSize.Y/2)).Unit * 1000 end
                            pData.Line2D.From, pData.Line2D.To, pData.Line2D.Visible = fromV, toV, true
                        end
                    end
                else removePlayerESP(op) end
            end
        end
    end
end)

local function createLoadingScreen()
    if not player:FindFirstChild("PlayerGui") then repeat task.wait() until player:FindFirstChild("PlayerGui") end
    local gui = Instance.new("ScreenGui", player.PlayerGui); gui.Name, gui.DisplayOrder = "Loading_ChienDo", 999
    local bg = Instance.new("Frame", gui); bg.Size, bg.BackgroundColor3, bg.BackgroundTransparency, bg.BorderSizePixel = UDim2.new(1,0,1,0), Color3.new(0,0,0), 0.2, 0
    local c = Instance.new("Frame", bg); c.Size, c.Position, c.BackgroundTransparency = UDim2.new(0,600,0,100), UDim2.new(0.5,-300,0.5,-50), 1
    local tc = Instance.new("Frame", c); tc.Size, tc.BackgroundTransparency = UDim2.new(1,0,0,60), 1
    local l = Instance.new("UIListLayout", tc); l.FillDirection, l.HorizontalAlignment, l.VerticalAlignment = Enum.FillDirection.Horizontal, Enum.HorizontalAlignment.Center, Enum.VerticalAlignment.Center
    local sub = Instance.new("TextLabel", c); sub.Size, sub.Position, sub.BackgroundTransparency, sub.Text, sub.TextColor3, sub.Font, sub.TextSize, sub.TextTransparency = UDim2.new(1,0,0,30), UDim2.new(0,0,0,65), 1, "Owner TikTok: chiendo_not_chiendo", Color3.fromRGB(180,180,180), Enum.Font.Gotham, 16, 1

    task.spawn(function()
        for i = 1, 120 do
            task.spawn(function()
                local dot = Instance.new("Frame", bg); dot.Size = UDim2.new(0, 5, 0, 5); dot.BackgroundColor3 = Color3.new(1, 0, 0); dot.BorderSizePixel = 0; dot.Position = UDim2.new(math.random(), 0, math.random(), 0); Instance.new("UICorner", dot)
                TweenService:Create(dot, TweenInfo.new(3, Enum.EasingStyle.Linear), {Position = UDim2.new(math.random(), 0, math.random(), 0), BackgroundTransparency = 1}):Play()
            end)
        end
    end)

    local labels = {}
    for _, code in utf8.codes("OBBY FOR UGC") do
        local lb = Instance.new("TextLabel", tc); lb.Size, lb.BackgroundTransparency, lb.Text, lb.TextColor3, lb.Font, lb.TextSize, lb.TextTransparency, lb.Position = UDim2.new(0, 26, 1, 0), 1, utf8.char(code), Color3.new(1,0,0), Enum.Font.GothamBold, 36, 1, UDim2.new(0,0,0,60)
        table.insert(labels, lb)
    end
    task.spawn(function()
        for _, lb in ipairs(labels) do TweenService:Create(lb, TweenInfo.new(0.4, Enum.EasingStyle.Back), {Position = UDim2.new(0,0,0,0), TextTransparency = 0}):Play(); task.wait(0.08) end
        TweenService:Create(sub, TweenInfo.new(0.5), {TextTransparency = 0}):Play()
    end)
    task.wait(3); gui:Destroy()
end

createLoadingScreen()

local afkGui = Instance.new("ScreenGui", player.PlayerGui); afkGui.Name, afkGui.ResetOnSpawn, afkGui.DisplayOrder, afkGui.IgnoreGuiInset = "AFK_FullScreen_Saver", false, 998, true
local afkScreen = Instance.new("Frame", afkGui); afkScreen.Size, afkScreen.BackgroundColor3, afkScreen.BorderSizePixel, afkScreen.Visible = UDim2.new(1,0,1,0), Color3.fromRGB(5,5,5), 0, false
local card = Instance.new("Frame", afkScreen); card.Size, card.Position, card.BackgroundColor3, card.BorderSizePixel = UDim2.new(0,280,0,190), UDim2.new(0.5,-140,0.5,-95), Color3.fromRGB(15,15,15), 0; Instance.new("UICorner", card)
Instance.new("UIStroke", card).Color = Color3.fromRGB(255,0,0)
local function createLbl(name, pos, col, txt) local lb = Instance.new("TextLabel", card); lb.Size, lb.Position, lb.BackgroundTransparency, lb.TextColor3, lb.Font, lb.TextSize, lb.TextXAlignment, lb.Text = UDim2.new(1,-20,0,22), pos, 1, col, Enum.Font.GothamMedium, 12, Enum.TextXAlignment.Left, txt; return lb end
local cTitle = Instance.new("TextLabel", card); cTitle.Size, cTitle.Position, cTitle.BackgroundTransparency, cTitle.Text, cTitle.TextColor3, cTitle.Font, cTitle.TextSize = UDim2.new(1,0,0,30), UDim2.new(0,0,0,8), 1, "🌙 AFK DASHBOARD", Color3.fromRGB(255,60,60), Enum.Font.GothamBold, 14
local timeLb, stageLb, gainedLb, scoreLb = createLbl("t", UDim2.new(0,15,0,42), Color3.fromRGB(220,220,220), "⏱️ Time AFK: 00h 00m 00s"), createLbl("s", UDim2.new(0,15,0,68), Color3.fromRGB(220,220,220), "📍 Current Stage: 0"), createLbl("g", UDim2.new(0,15,0,94), Color3.fromRGB(0,255,120), "📈 Stages Farmed: +0"), createLbl("sc", UDim2.new(0,15,0,120), Color3.fromRGB(255,200,0), "🏆 Current Score: 0")
local exitAfkBtn = Instance.new("TextButton", card); exitAfkBtn.Size, exitAfkBtn.Position, exitAfkBtn.BackgroundColor3, exitAfkBtn.Text, exitAfkBtn.TextColor3, exitAfkBtn.Font, exitAfkBtn.TextSize = UDim2.new(1,-30,0,28), UDim2.new(0,15,0,150), Color3.fromRGB(255,40,40), "EXIT AFK MODE", Color3.new(1,1,1), Enum.Font.GothamBold, 11; Instance.new("UICorner", exitAfkBtn)

local function getCurrentStageNum()
    local hrp = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
    local cpFolder = DEFAULT_SETTINGS.CHECKPOINT_FOLDER
    if not hrp or not cpFolder then return 0 end
    local closest, minDist = 0, math.huge
    for _, cp in pairs(cpFolder:GetChildren()) do
        local n = tonumber(cp.Name); if n and cp:IsA("BasePart") then local d = (hrp.Position - cp.Position).Magnitude; if d < minDist then minDist = d; closest = n end end
    end
    return closest
end

local function getLeaderstatScore()
    local ls = player:FindFirstChild("leaderstats")
    if ls then local obj = ls:FindFirstChild("Stage") or ls:FindFirstChild("Score") or ls:FindFirstChild("Points") or ls:FindFirstChild("UGC"); if obj then return obj.Value end end
    return 0
end

task.spawn(function()
    while afkGui.Parent do
        local curStage = getCurrentStageNum()
        if running and curStage > 0 and curStage % 50 == 0 and curStage ~= lastNotifiedMilestone then lastNotifiedMilestone = curStage; sendNotification("🎉 STAGE MILESTONE!", "Đạt Stage " .. tostring(curStage) .. "!") end
        if autoRebirthEnabled and running then
            if curStage == 251 then
                if stage251ReachedTime == 0 then stage251ReachedTime = tick() elseif tick() - stage251ReachedTime >= 10 then triggerRebirth(); stage251ReachedTime = tick() end
            else stage251ReachedTime = 0 end
        end
        if antiAfkEnabled and tick() - lastJumpTime >= 900 then
            lastJumpTime = tick()
            pcall(function() local h = player.Character and player.Character:FindFirstChildOfClass("Humanoid"); if h then h.Jump = true else VirtualUser:Button1Down(Vector2.new(0,0), camera.CFrame); VirtualUser:Button1Up(Vector2.new(0,0), camera.CFrame) end end)
        end
        if afkModeEnabled then
            local el = math.floor(tick() - afkStartTime)
            timeLb.Text = string.format("⏱️ Time AFK: %02dh %02dm %02ds", math.floor(el/3600), math.floor((el%3600)/60), el%60)
            stageLb.Text = "📍 Current Stage: " .. tostring(curStage)
            if curStage >= afkStartStage then stagesGained = curStage - afkStartStage end
            gainedLb.Text = "📈 Stages Farmed: +" .. tostring(stagesGained)
            scoreLb.Text = "🏆 Current Score: " .. tostring(getLeaderstatScore())
        end
        task.wait(1)
    end
end)

local mainGui = Instance.new("ScreenGui", player.PlayerGui); mainGui.Name, mainGui.ResetOnSpawn, mainGui.DisplayOrder = "AutoObby_GodAlways", false, 999
local frame = Instance.new("Frame", mainGui); frame.Size, frame.Position, frame.BackgroundColor3, frame.BorderSizePixel, frame.Active = UDim2.new(0,220,0,180), UDim2.new(0,30,0.5,-90), Color3.fromRGB(15,15,15), 0, true; Instance.new("UICorner", frame)

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

local function makeDraggable(g)
    local drag, dStart, sPos
    g.InputBegan:Connect(function(i) if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then drag, dStart, sPos = true, i.Position, g.Position end end)
    UserInputService.InputChanged:Connect(function(i) if drag and (i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch) then local d = i.Position - dStart; g.Position = UDim2.new(sPos.X.Scale, sPos.X.Offset + d.X, sPos.Y.Scale, sPos.Y.Offset + d.Y) end end)
    UserInputService.InputEnded:Connect(function(i) if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then drag = false end end)
end
makeDraggable(frame)

local settingsPanel = Instance.new("Frame", frame); settingsPanel.Size, settingsPanel.Position, settingsPanel.BackgroundColor3, settingsPanel.BorderSizePixel, settingsPanel.Visible, settingsPanel.ZIndex = UDim2.new(0,180,0,235), UDim2.new(1,8,0,0), Color3.fromRGB(15,15,15), 0, false, 4; Instance.new("UICorner", settingsPanel)

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

local settingsTitle = Instance.new("TextLabel", settingsPanel); settingsTitle.Size, settingsTitle.Position, settingsTitle.BackgroundTransparency, settingsTitle.Text, settingsTitle.TextColor3, settingsTitle.Font, settingsTitle.TextSize, settingsTitle.ZIndex = UDim2.new(1,0,0,24), UDim2.new(0,0,0,4), 1, "SETTINGS [v1.131 Fix]", Color3.new(1,1,1), Enum.Font.GothamBold, 12, 6

local speedTitle = Instance.new("TextLabel", settingsPanel); speedTitle.Size, speedTitle.Position, speedTitle.BackgroundTransparency, speedTitle.Text, speedTitle.TextColor3, speedTitle.Font, speedTitle.TextSize, speedTitle.TextXAlignment, speedTitle.ZIndex = UDim2.new(1,-20,0,16), UDim2.new(0,8,0,26), 1, "Fly Speed (20 - 70):", Color3.fromRGB(220,220,220), Enum.Font.Gotham, 10, Enum.TextXAlignment.Left, 6
local speedInput = Instance.new("TextBox", settingsPanel); speedInput.Size, speedInput.Position, speedInput.BackgroundColor3, speedInput.Text, speedInput.TextColor3, speedInput.Font, speedInput.TextSize, speedInput.ZIndex = UDim2.new(0,40,0,18), UDim2.new(1,-48,0,24), Color3.fromRGB(30,30,30), tostring(currentFlySpeed), Color3.new(1,0,0), Enum.Font.GothamBold, 10, 7; Instance.new("UICorner", speedInput)
local sliderBG = Instance.new("Frame", settingsPanel); sliderBG.Size, sliderBG.Position, sliderBG.BackgroundColor3, sliderBG.BorderSizePixel, sliderBG.ZIndex = UDim2.new(1,-16,0,5), UDim2.new(0,8,0,46), Color3.fromRGB(35,35,35), 0, 6; Instance.new("UICorner", sliderBG)
local sliderFill = Instance.new("Frame", sliderBG); sliderFill.Size, sliderFill.BackgroundColor3, sliderFill.BorderSizePixel, sliderFill.ZIndex = UDim2.new((currentFlySpeed-20)/50,0,1,0), Color3.new(1,0,0), 0, 7; Instance.new("UICorner", sliderFill)
local sliderBtn = Instance.new("TextButton", sliderBG); sliderBtn.Size, sliderBtn.BackgroundTransparency, sliderBtn.Text, sliderBtn.ZIndex = UDim2.new(1,0,1,0), 1, "", 8

local function updateSpeed(val)
    val = math.clamp(math.floor(val + 0.5), 20, 70); currentFlySpeed = val; speedInput.Text = tostring(val)
    sliderFill.Size = UDim2.new((val - 20) / 50, 0, 1, 0)
end
speedInput.FocusLost:Connect(function() local n = tonumber(speedInput.Text); if n then updateSpeed(n) else speedInput.Text = tostring(currentFlySpeed) end end)

sliderBtn.InputBegan:Connect(function(i) 
    if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then 
        isDraggingFly = true; isDraggingFc = false
        updateSpeed(20 + math.clamp((i.Position.X - sliderBG.AbsolutePosition.X) / sliderBG.AbsoluteSize.X, 0, 1) * 50) 
    end 
end)

local function createToggleBtn(yPos, textOn, textOff, defaultState, callback)
    local btn = Instance.new("TextButton", settingsPanel)
    btn.Size, btn.Position, btn.BackgroundTransparency, btn.Text, btn.TextColor3, btn.Font, btn.TextSize, btn.TextXAlignment, btn.ZIndex = UDim2.new(1,-16,0,20), UDim2.new(0,8,0,yPos), 1, defaultState and textOn or textOff, Color3.fromRGB(220,220,220), Enum.Font.Gotham, 10, Enum.TextXAlignment.Left, 7
    local state = defaultState
    btn.MouseButton1Click:Connect(function()
        state = not state
        btn.Text = state and textOn or textOff
        callback(state)
    end)
    return btn
end

local espBtn = createToggleBtn(58, "🔴 ESP Lines", "ESP Lines  OFF", espEnabled, function(s) espEnabled = s end)
local playerEspBtn = createToggleBtn(80, "🔵 Player ESP", "Player ESP  OFF", playerEspEnabled, function(s) playerEspEnabled = s end)
local playerHlBtn = createToggleBtn(102, "🔴 Player Highlight", "Player Highlight  OFF", playerHighlightEnabled, function(s) playerHighlightEnabled = s; updatePlayerHighlight() end)
local rebirthBtn = createToggleBtn(124, "🔴 Auto Rebirth", "Auto Rebirth  OFF", autoRebirthEnabled, function(s) autoRebirthEnabled = s end)
local antiAfkBtn = createToggleBtn(146, "🔴 Anti-AFK (15m)", "Anti-AFK  OFF", antiAfkEnabled, function(s) antiAfkEnabled = s end)

local freeCamBtn = Instance.new("TextButton", settingsPanel); freeCamBtn.Size, freeCamBtn.Position, freeCamBtn.BackgroundTransparency, freeCamBtn.Text, freeCamBtn.TextColor3, freeCamBtn.Font, freeCamBtn.TextSize, freeCamBtn.TextXAlignment, freeCamBtn.ZIndex = UDim2.new(1,-40,0,20), UDim2.new(0,8,0,168), 1, "Free Cam  OFF", Color3.fromRGB(220,220,220), Enum.Font.Gotham, 10, Enum.TextXAlignment.Left, 7
local fcArrowBtn = Instance.new("TextButton", settingsPanel); fcArrowBtn.Size, fcArrowBtn.Position, fcArrowBtn.BackgroundTransparency, fcArrowBtn.Text, fcArrowBtn.TextColor3, fcArrowBtn.Font, fcArrowBtn.TextSize, fcArrowBtn.ZIndex = UDim2.new(0,24,0,20), UDim2.new(1,-30,0,168), 1, "▼", Color3.fromRGB(200,200,200), Enum.Font.GothamBold, 10, 7

local fcSettingsContainer = Instance.new("Frame", settingsPanel); fcSettingsContainer.Size, fcSettingsContainer.Position, fcSettingsContainer.BackgroundTransparency, fcSettingsContainer.Visible, fcSettingsContainer.ZIndex = UDim2.new(1,-16,0,36), UDim2.new(0,8,0,190), 1, false, 7
local fcSpeedTitle = Instance.new("TextLabel", fcSettingsContainer); fcSpeedTitle.Size, fcSpeedTitle.Position, fcSpeedTitle.BackgroundTransparency, fcSpeedTitle.Text, fcSpeedTitle.TextColor3, fcSpeedTitle.Font, fcSpeedTitle.TextSize, fcSpeedTitle.TextXAlignment, fcSpeedTitle.ZIndex = UDim2.new(1,-35,0,14), UDim2.new(0,0,0,0), 1, "FC Speed (40-100): " .. tostring(freeCamSpeed), Color3.fromRGB(180,180,180), Enum.Font.Gotham, 9, Enum.TextXAlignment.Left, 7
local fcSliderBG = Instance.new("Frame", fcSettingsContainer); fcSliderBG.Size, fcSliderBG.Position, fcSliderBG.BackgroundColor3, fcSliderBG.BorderSizePixel, fcSliderBG.ZIndex = UDim2.new(1,0,0,5), UDim2.new(0,0,0,18), Color3.fromRGB(35,35,35), 0, 7; Instance.new("UICorner", fcSliderBG)
local fcSliderFill = Instance.new("Frame", fcSliderBG); fcSliderFill.Size, fcSliderFill.BackgroundColor3, fcSliderFill.BorderSizePixel, fcSliderFill.ZIndex = UDim2.new((freeCamSpeed-40)/60,0,1,0), Color3.fromRGB(0,150,255), 0, 8; Instance.new("UICorner", fcSliderFill)
local fcSliderBtn = Instance.new("TextButton", fcSliderBG); fcSliderBtn.Size, fcSliderBtn.BackgroundTransparency, fcSliderBtn.Text, fcSliderBtn.ZIndex = UDim2.new(1,0,1,0), 1, "", 9

local shaderBtn = createToggleBtn(190, "🔴 Shader V131", "Shader V131  OFF", shaderEnabled, function(s)
    shaderEnabled = s
    if shaderEnabled then enableShader() else disableShader() end
end)

local afkBtn = createToggleBtn(212, "🔴 AFK Mode", "AFK Mode  OFF", afkModeEnabled, function(s)
    afkModeEnabled = s
    afkScreen.Visible = afkModeEnabled
    if afkModeEnabled then
        afkStartTime, afkStartStage, stagesGained = tick(), getCurrentStageNum(), 0
        pcall(function() settings():GetService("RenderSettings").QualityLevel = Enum.QualityLevel.Level01; if set3drenderingenabled then set3drenderingenabled(false) end end)
    else
        pcall(function() settings():GetService("RenderSettings").QualityLevel = Enum.QualityLevel.Automatic; if set3drenderingenabled then set3drenderingenabled(true) end end)
    end
    updatePlayerHighlight()
end)

local fcSpeedOpen = false
fcArrowBtn.MouseButton1Click:Connect(function()
    fcSpeedOpen = not fcSpeedOpen
    fcSettingsContainer.Visible = fcSpeedOpen
    fcArrowBtn.Text = fcSpeedOpen and "▲" or "▼"
    if fcSpeedOpen then
        settingsPanel.Size = UDim2.new(0, 180, 0, 275)
        shaderBtn.Position = UDim2.new(0, 8, 0, 228)
        afkBtn.Position = UDim2.new(0, 8, 0, 250)
    else
        settingsPanel.Size = UDim2.new(0, 180, 0, 235)
        shaderBtn.Position = UDim2.new(0, 8, 0, 190)
        afkBtn.Position = UDim2.new(0, 8, 0, 212)
    end
end)

local function updateFcSpeed(val)
    val = math.clamp(math.floor(val + 0.5), 40, 100); freeCamSpeed = val; fcSpeedTitle.Text = "FC Speed (40-100): " .. tostring(val)
    fcSliderFill.Size = UDim2.new((val - 40) / 60, 0, 1, 0)
end

fcSliderBtn.InputBegan:Connect(function(i) 
    if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then 
        isDraggingFc = true; isDraggingFly = false
        updateFcSpeed(40 + math.clamp((i.Position.X - fcSliderBG.AbsolutePosition.X) / fcSliderBG.AbsoluteSize.X, 0, 1) * 60) 
    end 
end)

UserInputService.InputChanged:Connect(function(i) 
    if i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch then
        if isDraggingFly then
            updateSpeed(20 + math.clamp((i.Position.X - sliderBG.AbsolutePosition.X) / sliderBG.AbsoluteSize.X, 0, 1) * 50)
        elseif isDraggingFc then
            updateFcSpeed(40 + math.clamp((i.Position.X - fcSliderBG.AbsolutePosition.X) / fcSliderBG.AbsoluteSize.X, 0, 1) * 60)
        end
    end
end)

local function toggleFreeCam(state)
    freeCamEnabled = state; lockPlayerCharacter(freeCamEnabled)
    if freeCamEnabled then
        freeCamBtn.Text = "🔴 Free Cam"
        freeCamPos = camera.CFrame.Position; local _, y, _ = camera.CFrame:ToOrientation(); freeCamYaw, freeCamPitch = y, 0
    else freeCamBtn.Text = "Free Cam  OFF"; camera.CameraType = Enum.CameraType.Custom end
end
freeCamBtn.MouseButton1Click:Connect(function() toggleFreeCam(not freeCamEnabled) end)
exitAfkBtn.MouseButton1Click:Connect(function() afkModeEnabled = false; afkScreen.Visible = false; afkBtn.Text = "AFK Mode  OFF"; updatePlayerHighlight() end)

local topC = Instance.new("Frame", frame); topC.Size, topC.BackgroundTransparency, topC.ZIndex = UDim2.new(1,0,0,70), 1, 5
local function createTxt(pos, sz, txt, col, szFont) local lb = Instance.new("TextLabel", topC); lb.Size, lb.Position, lb.BackgroundTransparency, lb.Text, lb.TextColor3, lb.Font, lb.TextSize, lb.ZIndex = sz, pos, 1, txt, col, Enum.Font.Gotham, szFont, 6; return lb end
createTxt(UDim2.new(0,0,0,6), UDim2.new(1,0,0,20), "OBBY FOR UGC", Color3.new(1,1,1), 15).Font = Enum.Font.GothamBold
createTxt(UDim2.new(0,0,0,24), UDim2.new(1,0,0,16), "(AFK OR PLAY)", Color3.fromRGB(200,200,200), 11)
createTxt(UDim2.new(0,0,0,38), UDim2.new(1,0,0,26), "Owner\nTiktok : Chiendo_not_chiendo", Color3.fromRGB(160,160,160), 10)

local settingsBtn = Instance.new("TextButton", frame); settingsBtn.Size, settingsBtn.Position, settingsBtn.BackgroundTransparency, settingsBtn.Text, settingsBtn.TextColor3, settingsBtn.Font, settingsBtn.TextSize, settingsBtn.ZIndex = UDim2.new(0,25,0,25), UDim2.new(0,5,0,5), 1, "⚙️", Color3.new(1,1,1), Enum.Font.GothamBold, 16, 10
local infoBtn = Instance.new("TextButton", frame); infoBtn.Size, infoBtn.Position, infoBtn.BackgroundTransparency, infoBtn.Text, infoBtn.TextColor3, infoBtn.Font, infoBtn.TextSize, infoBtn.ZIndex = UDim2.new(0,25,0,25), UDim2.new(0,32,0,5), 1, "!", Color3.new(1,0,0), Enum.Font.GothamBold, 20, 10
settingsBtn.MouseButton1Click:Connect(function() isSettingsOpen = not isSettingsOpen; settingsPanel.Visible = isSettingsOpen end)

local bugFrame = Instance.new("TextButton", mainGui); bugFrame.Size, bugFrame.Position, bugFrame.BackgroundColor3, bugFrame.BackgroundTransparency, bugFrame.Visible, bugFrame.ZIndex = UDim2.new(0,320,0,180), UDim2.new(0.5,-160,0.5,-90), Color3.fromRGB(10,10,10), 0.25, false, 100; Instance.new("UICorner", bugFrame)
Instance.new("UIStroke", bugFrame).Color = Color3.new(1,0,0)
local bugLbl = Instance.new("TextLabel", bugFrame); bugLbl.Size, bugLbl.Position, bugLbl.BackgroundTransparency, bugLbl.Text, bugLbl.TextColor3, bugLbl.Font, bugLbl.TextSize, bugLbl.TextWrapped, bugLbl.ZIndex = UDim2.new(1,-20,1,-20), UDim2.new(0,10,0,10), 1, "-INFO BOARD-\n-Version 1.131 Fix Syntax-\n+ Fixed syntax typo error in protection toggle\n+ Smooth & soft daylight lighting\n+ Stable Execution on Delta Mobile", Color3.new(1,1,1), Enum.Font.GothamMedium, 12, true, 101
infoBtn.MouseButton1Click:Connect(function() bugFrame.Visible = not bugFrame.Visible end)
bugFrame.MouseButton1Click:Connect(function() bugFrame.Visible = false end)

local bottomC = Instance.new("Frame", frame); bottomC.Size, bottomC.Position, bottomC.BackgroundTransparency, bottomC.ZIndex = UDim2.new(1,0,0,110), UDim2.new(0,0,0,70), 1, 5
local btn = Instance.new("TextButton", bottomC); btn.Size, btn.Position, btn.BackgroundColor3, btn.Text, btn.TextColor3, btn.Font, btn.TextSize, btn.ZIndex, btn.Active = UDim2.new(0,180,0,50), UDim2.new(0.5,-90,0,20), Color3.new(1,1,1), "Auto Farm Stage: OFF", Color3.new(0,0,0), Enum.Font.GothamBold, 15, 6, true; Instance.new("UICorner", btn)

local toggleBtn = Instance.new("TextButton", frame); toggleBtn.Size, toggleBtn.Position, toggleBtn.BackgroundTransparency, toggleBtn.Text, toggleBtn.TextColor3, toggleBtn.Font, toggleBtn.TextSize, toggleBtn.ZIndex = UDim2.new(0,25,0,25), UDim2.new(1,-30,0,5), 1, "▲", Color3.new(1,1,1), Enum.Font.GothamBold, 18, 10
toggleBtn.MouseButton1Click:Connect(function()
    isMinimized = not isMinimized
    if isMinimized then frame:TweenSize(UDim2.new(0,220,0,70), "Out", "Quart", 0.3, true); toggleBtn.Text = "▼"; bottomC.Visible, settingsPanel.Visible = false, false
    else frame:TweenSize(UDim2.new(0,220,0,180), "Out", "Quart", 0.3, true); toggleBtn.Text = "▲"; bottomC.Visible = true; settingsPanel.Visible = isSettingsOpen end
end)

local function toggleProtection(state)
    if state then
        if not protectionConnection then
            protectionConnection = RunService.Stepped:Connect(function()
                if running and player.Character then
                    local h = player.Character:FindFirstChildOfClass("Humanoid")
                    if h then h.MaxHealth, h.Health = math.huge, math.huge; h:SetStateEnabled(Enum.HumanoidStateType.Dead, false) end
                    for _, p in pairs(player.Character:GetDescendants()) do if p:IsA("BasePart") then p.CanCollide, p.AssemblyLinearVelocity, p.AssemblyAngularVelocity = false, Vector3.zero, Vector3.zero end end
                end
            end)
        end
    else
        if protectionConnection then protectionConnection:Disconnect(); protectionConnection = nil end
    end
end

local function handleStage243Bypass(target)
    if isBypassing243 then return end; isBypassing243 = true
    local hrp = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
    if not hrp or not running then isBypassing243 = false; return end
    local targetCF = target and (target.CFrame * CFrame.new(0, 1.5, 0)) or CFrame.new(1761, 776.5, -704.5)
    local startCF, dist = hrp.CFrame, (hrp.CFrame.Position - targetCF.Position).Magnitude
    local dur, startT = dist / currentFlySpeed, tick()
    local conn; conn = RunService.RenderStepped:Connect(function()
        if not running or not hrp or not hrp.Parent then if conn then conn:Disconnect() end; return end
        local alpha = math.clamp((tick() - startT) / dur, 0, 1); hrp.CFrame = startCF:Lerp(targetCF, alpha)
        if alpha >= 1 then if conn then conn:Disconnect() end end
    end)
    while running and (tick() - startT) < dur do task.wait() end
    if conn then conn:Disconnect() end; if not running then isBypassing243 = false; return end
    hrp.CFrame = targetCF
    if firetouchinterest and target and target:IsA("BasePart") then firetouchinterest(hrp, target, 0); task.wait(0.05); firetouchinterest(hrp, target, 1) end
    task.wait(0.2)
    local pos243 = hrp.Position; local offCF = CFrame.new(pos243.X, pos243.Y, pos243.Z + 200)
    dur, startT = 200 / currentFlySpeed, tick()
    local connZ; connZ = RunService.RenderStepped:Connect(function()
        if not running or not hrp or not hrp.Parent then if connZ then connZ:Disconnect() end; return end
        local alpha = math.clamp((tick() - startT) / dur, 0, 1); hrp.CFrame = CFrame.new(pos243):Lerp(offCF, alpha)
        if alpha >= 1 then if connZ then connZ:Disconnect() end end
    end)
    while running and (tick() - startT) < dur do task.wait() end
    if connZ then connZ:Disconnect() end; if not running then isBypassing243 = false; return end
    hrp.CFrame = offCF; toggleProtection(false); task.wait(math.random(20, 30) / 10); if running then toggleProtection(true) end
    isBypassing243 = false
end

local function flyToTarget(target)
    currentTargetPart = target
    local hrp = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
    if not hrp or not target or not running then currentTargetPart = nil; return end
    if target.Name == "243" then handleStage243Bypass(target); currentTargetPart = nil; return end
    local startCF, targetCF = hrp.CFrame, target.CFrame * CFrame.new(0, 2, 0)
    local dur, startT = (startCF.Position - targetCF.Position).Magnitude / currentFlySpeed, tick()
    local conn; conn = RunService.RenderStepped:Connect(function()
        if not running or not hrp or not hrp.Parent then if conn then conn:Disconnect() end; return end
        local alpha = math.clamp((tick() - startT) / dur, 0, 1); hrp.CFrame = startCF:Lerp(targetCF, alpha)
        if alpha >= 1 then if conn then conn:Disconnect() end end
    end)
    while running and (tick() - startT) < dur do task.wait() end
    if conn then conn:Disconnect() end; if not running then currentTargetPart = nil; return end
    hrp.CFrame = target.CFrame * CFrame.new(0, 1.5, 0)
    if firetouchinterest and target:IsA("BasePart") then firetouchinterest(hrp, target, 0); task.wait(0.05); firetouchinterest(hrp, target, 1) end
    toggleProtection(false); task.wait(math.random(3, 5) / 10); currentTargetPart = nil; if running then toggleProtection(true) end
end

local function startFarming()
    running, wasRunningBeforeDeath = true, true; toggleProtection(true)
    btn.Text, btn.BackgroundColor3, btn.TextColor3 = "Auto Farm Stage: ON", Color3.new(1,0,0), Color3.new(1,1,1)
    if farmingTask then return end
    farmingTask = task.spawn(function()
        local hrp = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
        local cpFolder = DEFAULT_SETTINGS.CHECKPOINT_FOLDER
        if hrp and cpFolder then
            local cp243, cp242 = cpFolder:FindFirstChild("243"), cpFolder:FindFirstChild("242")
            if cp243 and cp242 and (hrp.Position - cp243.Position).Magnitude <= 25 then
                local startCF, targetCF = hrp.CFrame, cp242.CFrame * CFrame.new(0, 1.5, 0)
                local dur, startT = (startCF.Position - targetCF.Position).Magnitude / currentFlySpeed, tick()
                local conn; conn = RunService.RenderStepped:Connect(function()
                    if not running or not hrp then if conn then conn:Disconnect() end; return end
                    local alpha = math.clamp((tick() - startT) / dur, 0, 1); hrp.CFrame = startCF:Lerp(targetCF, alpha)
                    if alpha >= 1 then if conn then conn:Disconnect() end end
                end)
                while running and (tick() - startT) < dur do task.wait() end
                if conn then conn:Disconnect() end
                if running and hrp then hrp.CFrame = targetCF; if firetouchinterest and cp242:IsA("BasePart") then firetouchinterest(hrp, cp242, 0); task.wait(0.05); firetouchinterest(hrp, cp242, 1) end end
                task.wait(math.random(3, 5) / 10)
            end
        end
        local lastStage, stuckT, isSkipping = -1, 0, false
        while running do
            local h = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
            if not h then task.wait(0.5); continue end
            local curStage, minDist = -1, math.huge
            local cpFolderLoop = DEFAULT_SETTINGS.CHECKPOINT_FOLDER
            if cpFolderLoop then
                for _, cp in pairs(cpFolderLoop:GetChildren()) do
                    local n = tonumber(cp.Name); if n and cp:IsA("BasePart") then local d = (h.Position - cp.Position).Magnitude; if d < minDist then minDist = d; curStage = n end end
                end
            end
            if curStage == lastStage then
                stuckT = stuckT + 0.5
                if stuckT >= 15 and not isSkipping then
                    isSkipping = true; local nextObj = cpFolderLoop and cpFolderLoop:FindFirstChild(tostring(curStage + 1))
                    if nextObj then flyToTarget(nextObj) end
                    stuckT, isSkipping = 0, false; task.wait(0.5); continue
                end
            else lastStage, stuckT = curStage, 0 end
            local nextTarget = cpFolderLoop and cpFolderLoop:FindFirstChild(tostring(curStage + 1))
            if nextTarget then flyToTarget(nextTarget) else task.wait(0.5) end
        end
        farmingTask = nil
    end)
end

local function stopFarming()
    running, wasRunningBeforeDeath, currentTargetPart = false, false, nil; toggleProtection(false)
    btn.Text, btn.BackgroundColor3, btn.TextColor3 = "Auto Farm Stage: OFF", Color3.new(1,1,1), Color3.new(0,0,0)
end

btn.MouseButton1Click:Connect(function() if running then stopFarming() else startFarming() end end)
player.CharacterAdded:Connect(function(newChar)
    toggleProtection(false); currentTargetPart = nil
    if freeCamEnabled then lockPlayerCharacter(true) end
    if wasRunningBeforeDeath then
        task.spawn(function() newChar:WaitForChild("HumanoidRootPart", 9e9); task.wait(1); if wasRunningBeforeDeath then startFarming() end end)
    else btn.Text, btn.BackgroundColor3, btn.TextColor3 = "Auto Farm Stage: OFF", Color3.new(1,1,1), Color3.new(0,0,0) end
end)

if _G.IsAutoRejoin then
    _G.IsAutoRejoin = nil
    task.spawn(function() local char = player.Character or player.CharacterAdded:Wait(); char:WaitForChild("HumanoidRootPart", 9e9); task.wait(1.5); startFarming() end)
end
