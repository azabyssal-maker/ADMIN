-- ENRIQUE BLADE V1.6 - 增强版（预测格挡 + 稳定飞行 + 双Hook + 内存优化 + 混淆钥匙）

local Players = game:GetService("Players")
local RS = game:GetService("RunService")
local UIS = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")
local TweenService = game:GetService("TweenService")
local Stats = game:GetService("Stats")
local player = Players.LocalPlayer

-- ========== 配置 ==========
getgenv().ENRIQUE_BLADE = {
    Enabled = false,
    AutoParry = false,
    ParryDistance = 25,        -- 格挡触发距离
    PredictionTime = 0.2,      -- 预测时间（秒）
    SpamEnabled = false,
    NoHead = false,
    NoLeg = false,
    FlyEnabled = false,
    FlySpeed = 50,
    AntiLag = false,
    ShowBallSpeed = false,
    ShowPingFPS = false,
    WalkSpeed = 16,
}
local CFG = getgenv().ENRIQUE_BLADE

-- ========== 钥匙系统（XOR混淆 + 简单反篡改） ==========
local function xorDecrypt(encoded, key)
    local result = ""
    for i = 1, #encoded do
        result = result .. string.char(string.byte(encoded, i) ~ key)
        key = (key + 1) % 256
    end
    return result
end
local encodedPass = "\x4a\x7f\x71\x78\x7d\x68\x6f\x3d\x3f\x2e\x2d\x3c\x36\x3b\x2a\x3b\x3a\x2d\x3c\x36\x3b\x2a\x3b\x3a\x2d\x3c\x36\x3b\x2a\x3b\x3a"
local KeyPassword = xorDecrypt(encodedPass, 0x5A)

local KeyUI = nil
local authenticated = false

-- 反篡改：如果配置被恶意修改则禁用核心功能
local originalCFG = {}
for k, v in pairs(CFG) do originalCFG[k] = v end
local function antiTamper()
    for k, v in pairs(CFG) do
        if originalCFG[k] ~= v and k ~= "SpamEnabled" and k ~= "ShowBallSpeed" and k ~= "ShowPingFPS" then
            warn("[ENRIQUE] 配置被篡改，已禁用核心功能")
            CFG.Enabled = false
            CFG.AutoParry = false
            CFG.FlyEnabled = false
            return false
        end
    end
    return true
end

-- ========== 内存优化：弱表存储球锁 ==========
local ball_locks = setmetatable({}, {__mode = "k"})
task.spawn(function()
    while true do
        task.wait(30)
        for k, v in pairs(ball_locks) do
            if not k or not k.Parent then ball_locks[k] = nil end
        end
    end
end)

-- ========== 增强 Remote Hook（__index + __namecall 双保险） ==========
local remote, f_raw = nil, nil
local c = {nil, nil, nil, nil, nil, nil, nil}
local mt = getrawmetatable(game)
local oldIdx = mt.__index
local oldNamecall = mt.__namecall
setreadonly(mt, false)

mt.__index = newcclosure(function(self, key)
    if key == "FireServer" then
        return function(obj, ...)
            local a = {...}
            if #a == 7 and typeof(a[4]) == "CFrame" then
                remote, f_raw = obj, obj.FireServer
                for i=1,7 do c[i] = a[i] end
            end
            return oldIdx(self, key)(obj, ...)
        end
    end
    return oldIdx(self, key)
end)

mt.__namecall = newcclosure(function(self, ...)
    local method = getnamecallmethod()
    if method == "FireServer" then
        local a = {...}
        if #a == 7 and typeof(a[4]) == "CFrame" then
            remote, f_raw = self, self.FireServer
            for i=1,7 do c[i] = a[i] end
        end
    end
    return oldNamecall(self, ...)
end)

setreadonly(mt, true)

-- 兜底：如果一直没抓到，定期扫描
task.spawn(function()
    while true do
        task.wait(3)
        if not remote then
            for _, v in ipairs(getgc(true)) do
                if type(v) == "function" and debug.info(v, "n") == "FireServer" then
                    remote = v
                    f_raw = v
                    break
                end
            end
        end
    end
end)

-- ========== 增强自动格挡：预测 + 优先级排序 ==========
local lastParry = 0
local function isBallTargetingPlayer(ball)
    local target = ball:GetAttribute("target") or ball:GetAttribute("Target")
    if target == player.Name then return true end
    local col = ball.Color
    if col and col.R > 0.5 and col.G < 0.4 and col.B < 0.4 then return true end
    return false
end

local function getBallPriority(ball, hrpPos)
    local vel = ball.Velocity
    local predictPos = ball.Position + vel * CFG.PredictionTime
    local distance = (predictPos - hrpPos).Magnitude
    local timeToImpact = distance / (vel.Magnitude + 0.001)
    return timeToImpact, distance
end

RS.RenderStepped:Connect(function()
    if not CFG.Enabled or not CFG.AutoParry then return end
    if not remote or not f_raw then return end
    if not antiTamper() then return end

    local now = tick()
    if now - lastParry < 0.2 then return end

    local balls = Workspace:FindFirstChild("Balls") or Workspace:FindFirstChild("BaseBall")
    local char = player.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    if not hrp or not balls then return end

    local threats = {}
    for _, ball in pairs(balls:GetChildren()) do
        if ball and ball.Parent and isBallTargetingPlayer(ball) then
            local time, dist = getBallPriority(ball, hrp.Position)
            if dist <= CFG.ParryDistance then
                table.insert(threats, {ball = ball, time = time, dist = dist})
            end
        end
    end
    if #threats == 0 then return end
    table.sort(threats, function(a, b) return a.time < b.time end)
    local target = threats[1].ball
    if target then
        lastParry = now
        pcall(f_raw, remote, unpack(c))
    end
end)

-- ========== 改进飞行（BodyPosition + 面向控制，手机稳定） ==========
local flyBody = nil
local flyGyro = nil
RS.RenderStepped:Connect(function()
    local char = player.Character
    if not char then return end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    local hum = char:FindFirstChildOfClass("Humanoid")
    if CFG.FlyEnabled and hrp and hum then
        if not flyBody then
            flyBody = Instance.new("BodyPosition", hrp)
            flyBody.Name = "FlyBodyPos"
            flyBody.MaxForce = Vector3.new(1e5, 1e5, 1e5)
            flyBody.D = 500
            flyBody.P = 2000
            flyGyro = Instance.new("BodyGyro", hrp)
            flyGyro.Name = "FlyGyro"
            flyGyro.MaxTorque = Vector3.new(1e6, 1e6, 1e6)
            flyGyro.P = 5000
        end
        local move = hum.MoveDirection
        flyBody.Position = hrp.Position + move * CFG.FlySpeed / 30
        -- 面向相机方向
        local lookDir = Workspace.CurrentCamera.CFrame.LookVector
        flyGyro.CFrame = CFrame.new(hrp.Position, hrp.Position + lookDir)
        hum.PlatformStand = true
    else
        if flyBody then flyBody:Destroy(); flyBody = nil end
        if flyGyro then flyGyro:Destroy(); flyGyro = nil end
        if hum then hum.PlatformStand = false end
    end
end)

-- ========== 步行速度 ==========
RS.RenderStepped:Connect(function()
    if CFG.Enabled then
        local char = player.Character
        if char then
            local hum = char:FindFirstChildOfClass("Humanoid")
            if hum then hum.WalkSpeed = CFG.WalkSpeed end
        end
    end
end)

-- ========== 视觉（NoHead, NoLeg） ==========
RS.RenderStepped:Connect(function()
    local char = player.Character
    if not char then return end
    local head = char:FindFirstChild("Head")
    if head then
        head.Transparency = CFG.NoHead and 1 or 0
        for _, v in pairs(head:GetChildren()) do
            if v:IsA("Decal") then v.Transparency = CFG.NoHead and 1 or 0 end
        end
    end
    for _, name in pairs({"RightFoot","RightLowerLeg","RightUpperLeg","LeftFoot","LeftLowerLeg","LeftUpperLeg"}) do
        local p = char:FindFirstChild(name)
        if p then p.Transparency = CFG.NoLeg and 1 or 0 end
    end
end)

-- ========== AntiLag ==========
local antiLagConn = nil
local function removeEffects(v)
    pcall(function()
        if v:IsA("Part") or v:IsA("MeshPart") then
            local n = string.lower(v.Name)
            if n:find("shield") or n:find("bubble") or n:find("sphere") or n:find("block") then
                v.Transparency = 1
                v.Material = Enum.Material.SmoothPlastic
                if v:FindFirstChildOfClass("SpecialMesh") then v:FindFirstChildOfClass("SpecialMesh"):Destroy() end
            end
        end
        if v:IsA("ParticleEmitter") then v.Enabled = false; v.Rate = 0; v.Lifetime = NumberRange.new(0) end
        if v:IsA("Trail") then v.Enabled = false; v.Lifetime = 0 end
        if v:IsA("Highlight") then v.Enabled = false end
        if v:IsA("PointLight") or v:IsA("SpotLight") or v:IsA("SurfaceLight") then v.Enabled = false end
        if v:IsA("BloomEffect") or v:IsA("ColorCorrectionEffect") or v:IsA("SunRaysEffect") then v.Enabled = false end
    end)
end
local function StartAntiLag()
    if antiLagConn then antiLagConn:Disconnect() end
    for _,v in ipairs(Workspace:GetDescendants()) do removeEffects(v) end
    antiLagConn = Workspace.DescendantAdded:Connect(function(v) task.defer(function() removeEffects(v) end) end)
end
local function StopAntiLag()
    if antiLagConn then antiLagConn:Disconnect(); antiLagConn = nil end
end

-- ========== Spam（极速单包） ==========
local spamGui = Instance.new("ScreenGui", player.PlayerGui)
spamGui.Name = "SpamPanel"
spamGui.Enabled = false
local spamFrame = Instance.new("Frame", spamGui)
spamFrame.Size = UDim2.new(0,160,0,80)
spamFrame.Position = UDim2.new(1,-170,0.4,0)
spamFrame.BackgroundColor3 = Color3.new(0,0,0)
spamFrame.Draggable = true
local spamBtn = Instance.new("TextButton", spamFrame)
spamBtn.Size = UDim2.new(1,-20,1,-20)
spamBtn.Position = UDim2.new(0,10,0,10)
spamBtn.Text = "START SPAM"
spamBtn.BackgroundColor3 = Color3.new(0,0,0)
spamBtn.TextColor3 = Color3.fromRGB(0,255,100)
local spamThread = nil
local function startSpam()
    if spamThread then return end
    spamThread = task.spawn(function()
        while CFG.SpamEnabled do
            if remote and f_raw then
                pcall(f_raw, remote, unpack(c))
            end
            task.wait(0.002)
        end
        spamThread = nil
    end)
end
spamBtn.MouseButton1Click:Connect(function()
    CFG.SpamEnabled = not CFG.SpamEnabled
    spamBtn.Text = CFG.SpamEnabled and "STOP SPAM" or "START SPAM"
    if CFG.SpamEnabled then startSpam() end
end)

-- ========== Ball Speed UI（降频） ==========
local ballUI, speedLabel, maxLabel, lastUpdate = nil, nil, nil, 0
local maxSpd = 0
RS.RenderStepped:Connect(function()
    if CFG.ShowBallSpeed then
        if not ballUI then
            ballUI = Instance.new("ScreenGui", player.PlayerGui)
            ballUI.Name = "BallSpeed"
            local f = Instance.new("Frame", ballUI)
            f.Size = UDim2.new(0,140,0,75)
            f.Position = UDim2.new(0.01,0,0.15,0)
            f.BackgroundColor3 = Color3.fromRGB(10,10,15)
            Instance.new("UICorner", f).CornerRadius = UDim.new(0,10)
            speedLabel = Instance.new("TextLabel", f)
            speedLabel.Size = UDim2.new(1,0,0,28)
            speedLabel.Position = UDim2.new(0,0,0,25)
            speedLabel.Text = "0"
            speedLabel.TextColor3 = Color3.new(1,1,1)
            speedLabel.Font = Enum.Font.GothamBold
            speedLabel.TextSize = 20
            maxLabel = Instance.new("TextLabel", f)
            maxLabel.Size = UDim2.new(1,0,0,18)
            maxLabel.Position = UDim2.new(0,0,0,55)
            maxLabel.Text = "MAX: 0"
            maxLabel.TextColor3 = Color3.fromRGB(150,150,150)
        end
        ballUI.Enabled = true
        if tick() - lastUpdate >= 0.1 then
            lastUpdate = tick()
            local balls = Workspace:FindFirstChild("Balls") or Workspace:FindFirstChild("BaseBall")
            local cur = 0
            if balls then
                for _, b in pairs(balls:GetChildren()) do
                    local spd = b.Velocity.Magnitude
                    if isBallTargetingPlayer(b) then cur = math.floor(spd); break end
                    if spd > cur then cur = math.floor(spd) end
                end
            end
            if cur > maxSpd then maxSpd = cur end
            if not balls or not next(balls:GetChildren()) then maxSpd = 0 end
            speedLabel.Text = tostring(cur)
            maxLabel.Text = "MAX: " .. maxSpd
        end
    else
        if ballUI then ballUI.Enabled = false end
    end
end)

-- ========== Ping / FPS UI ==========
local pingUI, pingLabel, fpsLabel, lastFPSUpdate = nil, nil, nil, 0
local frameCount = 0
RS.RenderStepped:Connect(function()
    frameCount = frameCount + 1
    if CFG.ShowPingFPS then
        if not pingUI then
            pingUI = Instance.new("ScreenGui", player.PlayerGui)
            pingUI.Name = "PingFPS"
            local f = Instance.new("Frame", pingUI)
            f.Size = UDim2.new(0,120,0,50)
            f.Position = UDim2.new(0.01,0,0.05,0)
            f.BackgroundColor3 = Color3.fromRGB(10,10,15)
            Instance.new("UICorner", f).CornerRadius = UDim.new(0,10)
            pingLabel = Instance.new("TextLabel", f)
            pingLabel.Size = UDim2.new(1,0,0,20)
            pingLabel.Position = UDim2.new(0,0,0,5)
            pingLabel.Text = "PING: --"
            pingLabel.TextColor3 = Color3.new(1,1,1)
            fpsLabel = Instance.new("TextLabel", f)
            fpsLabel.Size = UDim2.new(1,0,0,20)
            fpsLabel.Position = UDim2.new(0,0,0,28)
            fpsLabel.Text = "FPS: --"
        end
        pingUI.Enabled = true
        if tick() - lastFPSUpdate >= 1 then
            lastFPSUpdate = tick()
            local fps = frameCount
            frameCount = 0
            local ping = math.floor(Stats.Network.ServerStatsItem["Data Ping"]:GetValue() or 0)
            pingLabel.Text = "PING: " .. ping .. " ms"
            fpsLabel.Text = "FPS: " .. fps
        end
    else
        if pingUI then pingUI.Enabled = false end
    end
end)

-- ========== 钥匙 UI ==========
local function CreateKeyUI()
    if KeyUI and KeyUI.Parent then KeyUI:Destroy() end
    KeyUI = Instance.new("ScreenGui", player.PlayerGui)
    KeyUI.Name = "ENRIQUE_KEY"
    KeyUI.ResetOnSpawn = false
    local bg = Instance.new("Frame", KeyUI)
    bg.Size = UDim2.new(0, 400, 0, 260)
    bg.Position = UDim2.new(0.5, -200, 0.5, -130)
    bg.BackgroundColor3 = Color3.fromRGB(15, 15, 20)
    Instance.new("UICorner", bg).CornerRadius = UDim.new(0, 12)
    local title = Instance.new("TextLabel", bg)
    title.Size = UDim2.new(1, 0, 0, 45)
    title.BackgroundColor3 = Color3.fromRGB(0, 255, 100)
    title.Text = "⚡ ENRIQUE V1.6 ⚡"
    title.TextColor3 = Color3.new(0, 0, 0)
    title.Font = Enum.Font.GothamBold
    title.TextSize = 18
    local inputBox = Instance.new("TextBox", bg)
    inputBox.Size = UDim2.new(0.75, 0, 0, 40)
    inputBox.Position = UDim2.new(0.125, 0, 0.35, 0)
    inputBox.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
    inputBox.PlaceholderText = "Enter key..."
    inputBox.TextColor3 = Color3.fromRGB(255, 255, 255)
    inputBox.Font = Enum.Font.Gotham
    local submit = Instance.new("TextButton", bg)
    submit.Size = UDim2.new(0.35, 0, 0, 40)
    submit.Position = UDim2.new(0.125, 0, 0.6, 0)
    submit.BackgroundColor3 = Color3.fromRGB(0, 255, 100)
    submit.Text = "UNLOCK"
    submit.TextColor3 = Color3.new(0, 0, 0)
    local cancel = Instance.new("TextButton", bg)
    cancel.Size = UDim2.new(0.35, 0, 0, 40)
    cancel.Position = UDim2.new(0.525, 0, 0.6, 0)
    cancel.BackgroundColor3 = Color3.fromRGB(60, 60, 70)
    cancel.Text = "CANCEL"
    cancel.TextColor3 = Color3.fromRGB(255, 255, 255)
    local status = Instance.new("TextLabel", bg)
    status.Size = UDim2.new(0.9, 0, 0, 25)
    status.Position = UDim2.new(0.05, 0, 0.82, 0)
    status.BackgroundTransparency = 1
    status.Text = "Waiting for key..."
    status.TextColor3 = Color3.fromRGB(200,200,200)
    submit.MouseButton1Click:Connect(function()
        if inputBox.Text == KeyPassword then
            status.Text = "✓ Access granted!"
            status.TextColor3 = Color3.fromRGB(100,255,100)
            task.wait(0.5)
            KeyUI:Destroy()
            showMainUI()
        else
            status.Text = "✗ Invalid key!"
            status.TextColor3 = Color3.fromRGB(255,100,100)
            inputBox.Text = ""
        end
    end)
    cancel.MouseButton1Click:Connect(function()
        KeyUI:Destroy()
    end)
end

-- ========== 主 GUI ==========
local mainGUI = nil
function showMainUI()
    if mainGUI and mainGUI.Parent then mainGUI.Visible = true; return end
    mainGUI = Instance.new("ScreenGui", player.PlayerGui)
    mainGUI.Name = "ENRIQUE_GUI"
    mainGUI.ResetOnSpawn = false

    local Main = Instance.new("Frame", mainGUI)
    Main.Size = UDim2.new(0, 280, 0, 450)
    Main.Position = UDim2.new(0.5, -140, 0.5, -225)
    Main.BackgroundColor3 = Color3.fromRGB(15,15,15)
    Main.Draggable = true
    Instance.new("UICorner", Main).CornerRadius = UDim.new(0,10)
    local titleBar = Instance.new("Frame", Main)
    titleBar.Size = UDim2.new(1,0,0,35)
    titleBar.BackgroundColor3 = Color3.fromRGB(0,255,100)
    local title = Instance.new("TextLabel", titleBar)
    title.Size = UDim2.new(1,0,1,0)
    title.Text = "ENRIQUE BLADE V1.6"
    title.TextColor3 = Color3.new(0,0,0)
    title.Font = Enum.Font.GothamBold
    local minBtn = Instance.new("TextButton", titleBar)
    minBtn.Size = UDim2.new(0,28,0,28)
    minBtn.Position = UDim2.new(1,-32,0.5,-14)
    minBtn.Text = "-"
    minBtn.BackgroundColor3 = Color3.fromRGB(15,15,15)
    minBtn.TextColor3 = Color3.fromRGB(0,255,100)
    local miniBtn = Instance.new("TextButton", mainGUI)
    miniBtn.Size = UDim2.new(0,45,0,45)
    miniBtn.Position = UDim2.new(0,15,0.5,-22)
    miniBtn.Text = "E"
    miniBtn.BackgroundColor3 = Color3.fromRGB(0,255,100)
    miniBtn.Visible = false
    minBtn.MouseButton1Click:Connect(function() Main.Visible = false; miniBtn.Visible = true end)
    miniBtn.MouseButton1Click:Connect(function() Main.Visible = true; miniBtn.Visible = false end)

    local tabBar = Instance.new("Frame", Main)
    tabBar.Size = UDim2.new(1,0,0,35)
    tabBar.Position = UDim2.new(0,0,0,35)
    tabBar.BackgroundColor3 = Color3.fromRGB(22,22,22)
    local playerTab = Instance.new("TextButton", tabBar)
    playerTab.Size = UDim2.new(0.5,-1,1,-2)
    playerTab.Position = UDim2.new(0,0,0,1)
    playerTab.Text = "PLAYER"
    playerTab.BackgroundColor3 = Color3.fromRGB(35,35,35)
    local blatantTab = Instance.new("TextButton", tabBar)
    blatantTab.Size = UDim2.new(0.5,-1,1,-2)
    blatantTab.Position = UDim2.new(0.5,1,0,1)
    blatantTab.Text = "BLATANT"
    blatantTab.BackgroundColor3 = Color3.fromRGB(0,255,100)
    blatantTab.TextColor3 = Color3.new(0,0,0)

    local content = Instance.new("ScrollingFrame", Main)
    content.Size = UDim2.new(1,-10,1,-80)
    content.Position = UDim2.new(0,5,0,70)
    content.BackgroundTransparency = 1
    content.ScrollBarThickness = 4
    content.AutomaticCanvasSize = Enum.AutomaticSize.Y

    -- Player页面
    local playerPage = Instance.new("Frame", content)
    playerPage.Size = UDim2.new(1,0,0,0)
    playerPage.BackgroundTransparency = 1
    playerPage.Visible = false
    local playerLayout = Instance.new("UIListLayout", playerPage)
    playerLayout.Padding = UDim.new(0,10)

    -- 速度滑块
    local speedFrame = Instance.new("Frame", playerPage)
    speedFrame.Size = UDim2.new(1,-10,0,75)
    speedFrame.BackgroundColor3 = Color3.fromRGB(22,22,22)
    Instance.new("UICorner", speedFrame).CornerRadius = UDim.new(0,8)
    local speedLabel = Instance.new("TextLabel", speedFrame)
    speedLabel.Size = UDim2.new(0.5,0,0.35,0)
    speedLabel.Position = UDim2.new(0,12,0,5)
    speedLabel.Text = "Walk Speed: " .. CFG.WalkSpeed
    speedLabel.TextColor3 = Color3.fromRGB(220,220,220)
    local speedInput = Instance.new("TextBox", speedFrame)
    speedInput.Size = UDim2.new(0.2,0,0.35,0)
    speedInput.Position = UDim2.new(0.75,0,0,5)
    speedInput.BackgroundColor3 = Color3.fromRGB(40,40,45)
    speedInput.Text = tostring(CFG.WalkSpeed)
    local sliderBg = Instance.new("Frame", speedFrame)
    sliderBg.Size = UDim2.new(0.65,0,0,6)
    sliderBg.Position = UDim2.new(0.05,0,0.65,0)
    sliderBg.BackgroundColor3 = Color3.fromRGB(50,50,50)
    Instance.new("UICorner", sliderBg).CornerRadius = UDim.new(1,0)
    local fill = Instance.new("Frame", sliderBg)
    fill.Size = UDim2.new((CFG.WalkSpeed-16)/104,0,1,0)
    fill.BackgroundColor3 = Color3.fromRGB(255,100,0)
    local knob = Instance.new("Frame", sliderBg)
    knob.Size = UDim2.new(0,12,0,12)
    knob.Position = UDim2.new((CFG.WalkSpeed-16)/104,-6,0.5,-6)
    knob.BackgroundColor3 = Color3.fromRGB(255,150,50)
    local dragging = false
    local function updateSpeed(v)
        v = math.clamp(math.floor(v),16,120)
        CFG.WalkSpeed = v
        speedLabel.Text = "Walk Speed: " .. v
        speedInput.Text = tostring(v)
        local rel = (v-16)/104
        fill.Size = UDim2.new(rel,0,1,0)
        knob.Position = UDim2.new(rel,-6,0.5,-6)
    end
    sliderBg.InputBegan:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = true
            local rel = (i.Position.X - sliderBg.AbsolutePosition.X)/sliderBg.AbsoluteSize.X
            updateSpeed(16 + math.clamp(rel,0,1)*104)
        end
    end)
    UIS.InputChanged:Connect(function(i)
        if dragging and i.UserInputType == Enum.UserInputType.MouseMovement then
            local rel = (i.Position.X - sliderBg.AbsolutePosition.X)/sliderBg.AbsoluteSize.X
            updateSpeed(16 + math.clamp(rel,0,1)*104)
        end
    end)
    UIS.InputEnded:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1 then dragging = false end
    end)
    speedInput.FocusLost:Connect(function()
        local v = tonumber(speedInput.Text)
        if v then updateSpeed(v) else speedInput.Text = tostring(CFG.WalkSpeed) end
    end)

    -- Blatant页面
    local blatantPage = Instance.new("Frame", content)
    blatantPage.Size = UDim2.new(1,0,0,0)
    blatantPage.BackgroundTransparency = 1
    local blatantLayout = Instance.new("UIListLayout", blatantPage)
    blatantLayout.Padding = UDim.new(0,6)

    local function addToggle(text, key)
        local btn = Instance.new("TextButton", blatantPage)
        btn.Size = UDim2.new(1,-10,0,34)
        btn.BackgroundColor3 = Color3.fromRGB(28,28,28)
        btn.Text = text .. ": OFF"
        btn.TextColor3 = Color3.new(1,1,1)
        btn.Font = Enum.Font.GothamBold
        Instance.new("UICorner", btn).CornerRadius = UDim.new(0,6)
        btn.MouseButton1Click:Connect(function()
            CFG[key] = not CFG[key]
            btn.Text = text .. ": " .. (CFG[key] and "ON" or "OFF")
            btn.BackgroundColor3 = CFG[key] and Color3.fromRGB(0,255,100) or Color3.fromRGB(28,28,28)
            btn.TextColor3 = CFG[key] and Color3.new(0,0,0) or Color3.new(1,1,1)
            if key == "Enabled" then spamGui.Enabled = CFG.Enabled end
            if key == "AntiLag" then
                if CFG.AntiLag then StartAntiLag() else StopAntiLag() end
            end
        end)
        if CFG[key] then
            btn.BackgroundColor3 = Color3.fromRGB(0,255,100)
            btn.TextColor3 = Color3.new(0,0,0)
            btn.Text = text .. ": ON"
            if key == "AntiLag" then StartAntiLag() end
        end
    end

    addToggle("Main Power", "Enabled")
    addToggle("AUTO PARRY", "AutoParry")
    addToggle("FLY MODE", "FlyEnabled")
    addToggle("No Head", "NoHead")
    addToggle("No Leg", "NoLeg")
    addToggle("ANTI LAG", "AntiLag")
    addToggle("BALL SPEED", "ShowBallSpeed")
    addToggle("PING / FPS", "ShowPingFPS")

    -- 标签切换
    playerTab.MouseButton1Click:Connect(function()
        playerPage.Visible = true
        blatantPage.Visible = false
        playerTab.BackgroundColor3 = Color3.fromRGB(0,255,100)
        playerTab.TextColor3 = Color3.new(0,0,0)
        blatantTab.BackgroundColor3 = Color3.fromRGB(35,35,35)
        blatantTab.TextColor3 = Color3.fromRGB(200,200,200)
        content.CanvasSize = UDim2.new(0,0,0, playerLayout.AbsoluteContentSize.Y + 20)
    end)
    blatantTab.MouseButton1Click:Connect(function()
        playerPage.Visible = false
        blatantPage.Visible = true
        blatantTab.BackgroundColor3 = Color3.fromRGB(0,255,100)
        blatantTab.TextColor3 = Color3.new(0,0,0)
        playerTab.BackgroundColor3 = Color3.fromRGB(35,35,35)
        playerTab.TextColor3 = Color3.fromRGB(200,200,200)
        content.CanvasSize = UDim2.new(0,0,0, blatantLayout.AbsoluteContentSize.Y + 20)
    end)

    task.wait(0.1)
    content.CanvasSize = UDim2.new(0,0,0, blatantLayout.AbsoluteContentSize.Y + 20)
end

-- ========== 启动 ==========
local intro = Instance.new("ScreenGui", player.PlayerGui)
intro.Name = "Intro"
local black = Instance.new("Frame", intro)
black.Size = UDim2.new(1,0,1,0)
black.BackgroundColor3 = Color3.fromRGB(8,5,18)
local eLabel = Instance.new("TextLabel", black)
eLabel.Size = UDim2.new(0,120,0,120)
eLabel.Position = UDim2.new(0.5,-60,0.4,-60)
eLabel.BackgroundTransparency = 1
eLabel.Text = "E"
eLabel.TextColor3 = Color3.fromRGB(0,255,100)
eLabel.TextSize = 100
eLabel.Font = Enum.Font.GothamBold
local nameL = Instance.new("TextLabel", black)
nameL.Size = UDim2.new(0,500,0,60)
nameL.Position = UDim2.new(0.5,-250,0.5,30)
nameL.BackgroundTransparency = 1
nameL.Text = "ENRIQUE BLADE V1.6"
nameL.TextColor3 = Color3.new(1,1,1)
nameL.TextSize = 35
nameL.Font = Enum.Font.GothamBold
TweenService:Create(eLabel, TweenInfo.new(1.5, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true), {Rotation = 360}):Play()
task.wait(1.5)
TweenService:Create(black, TweenInfo.new(0.5), {BackgroundTransparency = 1}):Play()
task.wait(0.5)
intro:Destroy()
CreateKeyUI()
