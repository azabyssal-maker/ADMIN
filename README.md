-- Universal Admin - Full Version with hookfunction
-- Works on: Synapse, Krnl, Scriptware, Delta, Arceus X, Codex, etc.
repeat task.wait() until game:IsLoaded() and game:GetService("Players").LocalPlayer

local Players = game:GetService("Players")
local LP = Players.LocalPlayer
local RS = game:GetService("ReplicatedStorage")
local SG = game:GetService("StarterGui")
local UI = game:GetService("UserInputService")
local WS = game:GetService("Workspace")
local LG = game:GetService("Lighting")
local TS = game:GetService("TweenService")
local RunS = game:GetService("RunService")
local DS = game:GetService("Debris")
local Camera = WS.CurrentCamera

local PREFIX = ";"
local Cmds = {}
local CmdHistory = {}
local CmdIdx = 0
local Cooldowns = {}

-- ===== RANK SYSTEM =====
local Ranks = {
    Owner    = {Lv=5, Color=Color3.fromRGB(255,50,50),    Tag="[OWNER]"},
    Admin    = {Lv=4, Color=Color3.fromRGB(255,140,0),    Tag="[ADMIN]"},
    Moderator= {Lv=3, Color=Color3.fromRGB(255,200,0),    Tag="[MOD]"},
    Helper   = {Lv=2, Color=Color3.fromRGB(0,150,255),    Tag="[HELPER]"},
    Vip      = {Lv=1, Color=Color3.fromRGB(180,100,255),  Tag="[VIP]"},
    Guest    = {Lv=0, Color=Color3.fromRGB(200,200,200),  Tag=""},
}

-- Data store (persistent via writefile if available)
local Data = {RankData={}}
pcall(function()
    local d = readfile and readfile("admin_ranks.json")
    if d and d~="" then Data = game:GetService("HttpService"):JSONDecode(d) end
end)
Data.RankData = Data.RankData or {}
Data.RankData[tostring(LP.UserId)] = "Owner"

local function saveData()
    pcall(function()
        if writefile then
            writefile("admin_ranks.json", game:GetService("HttpService"):JSONEncode(Data))
        end
    end)
end

local function getRank(p)
    local r = Data.RankData[tostring(p.UserId)]
    return r and Ranks[r] and r or "Guest"
end
local function getLv(p)
    local r = Ranks[getRank(p)]
    return r and r.Lv or 0
end
local function hasPerm(p, rank)
    local req = Ranks[rank] and Ranks[rank].Lv or 999
    return getLv(p) >= req
end

-- ===== UTILITY =====
local function nfy(t, tt, d)
    pcall(function() SG:SetCore("SendNotification", {Title=t or "", Text=tt or "", Duration=d or 4}) end)
end
local function ac(msg)
    print("[Admin] "..msg)
    pcall(function() SG:SetCore("ChatMakeSystemMessage", {Text="[Admin] "..msg, Color=Color3.fromRGB(255,200,0)}) end)
end

local function findP(i)
    if not i or i=="" then return nil end
    local e = Players:FindFirstChild(i)
    if e then return e end
    for _, p in pairs(Players:GetPlayers()) do
        if p.Name:lower():sub(1, #i) == i:lower() or p.DisplayName:lower():sub(1, #i) == i:lower() then
            return p
        end
    end
    return nil
end
local function findPs(i)
    if not i or i=="" then return {} end
    if i:lower()=="all" then local r={}; for _,p in pairs(Players:GetPlayers()) do r[#r+1]=p end; return r end
    if i:lower()=="others" then local r={}; for _,p in pairs(Players:GetPlayers()) do if p~=LP then r[#r+1]=p end end; return r end
    if i:lower()=="admins" then local r={}; for _,p in pairs(Players:GetPlayers()) do if getLv(p)>=3 then r[#r+1]=p end end; return r end
    local p = findP(i)
    return p and {p} or {}
end

-- ===== COMMAND SYSTEM =====
local function reg(n, d)
    d.Name = n; d.Aliases = d.Aliases or {}; d.Rank = d.Rank or "Guest"
    d.Desc = d.Desc or ""; d.Args = d.Args or ""
    Cmds[n] = d
    for _, a in ipairs(d.Aliases) do Cmds[a] = d end
end

-- ===== HELP & INFO =====
reg("cmds", {Rank="Guest", Desc="显示指令列表", Func=function()
    local m = "===== 指令列表 (;"..PREFIX.."开头) =====\n"
    local cats = {["Management"]="管理",["Moderation"]="处罚",["Teleport"]="传送",
                  ["Player"]="玩家",["World"]="世界",["Fun"]="娱乐",["Utility"]="工具"}
    for _, cn in pairs(cats) do
        local has = false
        for n,c in pairs(Cmds) do
            if c.Category and cn and hasPerm(LP, c.Rank) then
                if not has then m = m.."\n--- "..cn.." ---\n"; has = true end
            end
        end
    end
    for n,c in pairs(Cmds) do
        if hasPerm(LP, c.Rank) and Cmds[n].Name == n then
            m = m.."  "..PREFIX..n.." "..c.Args.."  → "..c.Desc.."\n"
        end
    end
    m = m.."\n我的权限: "..getRank(LP).." (Lv."..getLv(LP)..")"
    ac(m)
end})
reg("myrank", {Rank="Guest", Desc="查看权限等级", Func=function()
    ac("权限: "..getRank(LP).." (Lv."..getLv(LP)..")")
end})
reg("players", {Rank="Guest", Desc="在线玩家列表", Func=function()
    local m = "在线玩家 ("..#Players:GetPlayers().."):\n"
    for _, p in pairs(Players:GetPlayers()) do
        m = m.."  "..p.Name.." ["..getRank(p).." Lv."..getLv(p).."]\n"
    end
    ac(m)
end})
reg("playerinfo", {Rank="Helper", Desc="查看玩家信息", Args="[玩家]", Func=function(_, a)
    local t = LP; if a[1] then t = findP(a[1]) or (ac("找不到"); return) end
    ac(t.Name.."\n  ID: "..t.UserId.."\n  权限: "..getRank(t).." (Lv."..getLv(t)..")\n  Ping: "..math.floor(t:GetNetworkPing()*1000).."ms")
end})
reg("serverinfo", {Rank="Helper", Desc="服务器信息", Func=function()
    ac("Place: "..game.PlaceId.."\nJob: "..game.JobId.."\n在线: "..#Players:GetPlayers().."/"..Players.MaxPlayers)
end})
reg("ping", {Rank="Guest", Desc="查看延迟", Func=function()
    ac("Ping: "..math.floor(LP:GetNetworkPing()*1000).."ms")
end})
reg("clear", {Rank="Moderator", Desc="清屏", Func=function()
    for i=1,50 do print("") end
end})

-- ===== MANAGEMENT =====
reg("setrank", {Rank="Owner", Category="Management", Desc="设置玩家权限", Args="<玩家> <等级>",
    Func=function(_, a)
        if #a<2 then return ac("用法: ;setrank <玩家> Owner/Admin/Moderator/Helper/Vip/Guest") end
        local t = findP(a[1]); if not t then return ac("找不到玩家") end
        local rn = a[2]:sub(1,1):upper()..a[2]:lower():sub(2)
        if not Ranks[rn] then return ac("无效等级。可选: Owner, Admin, Moderator, Helper, Vip, Guest") end
        Data.RankData[tostring(t.UserId)] = rn
        saveData()
        ac("✅ "..t.Name.." → "..rn)
        nfy("权限变更", t.Name.." 的权限已设为 "..rn, 4)
end})
reg("ranks", {Rank="Owner", Category="Management", Desc="查看所有权限等级", Func=function()
    local m = "权限等级:\n"
    for rn, rd in pairs(Ranks) do
        m = m.."  Lv."..rd.Lv.." "..rn.."\n"
    end
    ac(m)
end})

-- ===== MODERATION =====
reg("kick", {Rank="Moderator", Category="Moderation", Desc="踢出玩家", Args="<玩家> [原因]",
    Func=function(_, a)
        if #a<1 then return ac("用法: ;kick <玩家> [原因]") end
        local t = findP(a[1]); if not t then return ac("找不到玩家") end
        local r = table.concat(a, " ", 2) or "无原因"
        pcall(function() game:GetService("TeleportService"):Teleport(0, t) end)
        ac("✅ 已踢出 "..t.Name)
end})
reg("ban", {Rank="Admin", Category="Moderation", Desc="封禁玩家 (0=永久)", Args="<玩家> [分钟] [原因]",
    Func=function(_, a)
        if #a<1 then return ac("用法: ;ban <玩家> [分钟] [原因]") end
        local t = findP(a[1]); if not t then return ac("找不到玩家") end
        local dur = (tonumber(a[2]) or 0) * 60
        local ri = (tonumber(a[2]) and 3) or 2
        local r = table.concat(a, " ", ri) or "无原因"
        Data.BanData = Data.BanData or {}
        Data.BanData[tostring(t.UserId)] = {Reason=r, Time=os.time(), Expiry=dur>0 and os.time()+dur or 0}
        saveData()
        local ds = dur>0 and (math.floor(dur/60).."分钟") or "永久"
        ac("✅ 已封禁 "..t.Name.." | "..ds.." | "..r)
end})
reg("unban", {Rank="Admin", Category="Moderation", Desc="解封玩家", Args="<用户ID>",
    Func=function(_, a)
        if #a<1 then return ac("用法: ;unban <用户ID>") end
        local uid = tonumber(a[1]); if not uid then return ac("无效ID") end
        Data.BanData = Data.BanData or {}; Data.BanData[tostring(uid)] = nil; saveData()
        ac("✅ 已解封 ID: "..uid)
end})
reg("warn", {Rank="Moderator", Category="Moderation", Desc="警告玩家", Args="<玩家> [原因]",
    Func=function(_, a)
        if #a<1 then return ac("用法: ;warn <玩家> [原因]") end
        local t = findP(a[1]); if not t then return ac("找不到玩家") end
        local r = table.concat(a, " ", 2) or "无原因"
        Data.WarnData = Data.WarnData or {}
        local ws = Data.WarnData[tostring(t.UserId)] or {}
        table.insert(ws, {Reason=r, Time=os.time()})
        Data.WarnData[tostring(t.UserId)] = ws; saveData()
        nfy("⚠️ 警告", "来自 "..LP.Name..": "..r.." (第"..#ws.."次)", 6)
        ac("✅ 已警告 "..t.Name.." | "..r.." (#"..#ws..")")
end})
reg("warns", {Rank="Moderator", Category="Moderation", Desc="查看警告记录", Args="[玩家]",
    Func=function(_, a)
        local t = LP; if a[1] then t = findP(a[1]) or (ac("找不到"); return) end
        local ws = Data.WarnData and Data.WarnData[tostring(t.UserId)] or {}
        local m = t.Name.." 的警告 ("..#ws.."次):\n"
        for i, w in ipairs(ws) do m = m.."  #"..i.." "..w.Reason.." ("..os.date("%m/%d",w.Time)..")\n"; if i>=5 then break end end
        ac(m)
end})
reg("mute", {Rank="Moderator", Category="Moderation", Desc="禁言玩家", Args="<玩家> [分钟] [原因]",
    Func=function(_, a)
        if #a<1 then return ac("用法: ;mute <玩家> [分钟] [原因]") end
        local t = findP(a[1]); if not t then return ac("找不到玩家") end
        local dur = (tonumber(a[2]) or 0) * 60
        local ri = (tonumber(a[2]) and 3) or 2; local r = table.concat(a, " ", ri) or "无原因"
        Data.MuteData = Data.MuteData or {}
        Data.MuteData[tostring(t.UserId)] = {Reason=r, Expiry=dur>0 and os.time()+dur or 0}; saveData()
        local ds = dur>0 and (math.floor(dur/60).."分") or "永久"
        ac("✅ 已禁言 "..t.Name.." | "..ds)
end})
reg("unmute", {Rank="Moderator", Category="Moderation", Desc="解除禁言", Args="<玩家>",
    Func=function(_, a)
        if #a<1 then return ac("用法: ;unmute <玩家>") end
        local t = findP(a[1]); if not t then return ac("找不到") end
        Data.MuteData = Data.MuteData or {}; Data.MuteData[tostring(t.UserId)] = nil; saveData()
        ac("✅ 已解除 "..t.Name)
end})

-- ===== TELEPORT =====
reg("tp", {Rank="Moderator", Category="Teleport", Desc="传送到玩家", Args="<玩家>",
    Func=function(_, a)
        if #a<1 then return ac("用法: ;tp <玩家>") end
        local t = findP(a[1])
        if not t or not t.Character or not t.Character:FindFirstChild("HumanoidRootPart") then
            return ac("目标无效")
        end
        if LP.Character and LP.Character:FindFirstChild("HumanoidRootPart") then
            LP.Character.HumanoidRootPart.CFrame = t.Character.HumanoidRootPart.CFrame * CFrame.new(0,3,0)
            ac("✅ 已传送")
        end
end})
reg("bring", {Rank="Admin", Category="Teleport", Desc="拉取玩家到身边", Args="<玩家/all>",
    Func=function(_, a)
        if #a<1 then return ac("用法: ;bring <玩家/all>") end
        local ts = findPs(a[1])
        if not LP.Character or not LP.Character:FindFirstChild("HumanoidRootPart") then return end
        local cf = LP.Character.HumanoidRootPart.CFrame; local n = 0
        for _, t in ipairs(ts) do
            if t~=LP and t.Character and t.Character:FindFirstChild("HumanoidRootPart") then
                t.Character.HumanoidRootPart.CFrame = cf + Vector3.new(math.random(-3,3), 2, math.random(-3,3))
                n = n+1
            end
        end
        ac("✅ 拉了 "..n.." 人")
end})
reg("tploc", {Rank="Moderator", Category="Teleport", Desc="传送到坐标", Args="<x> <y> <z>",
    Func=function(_, a)
        if #a<3 then return ac("用法: ;tploc <x> <y> <z>") end
        local x,y,z = tonumber(a[1]), tonumber(a[2]), tonumber(a[3])
        if not (x and y and z) then return ac("坐标需要数字") end
        if LP.Character and LP.Character:FindFirstChild("HumanoidRootPart") then
            LP.Character.HumanoidRootPart.CFrame = CFrame.new(x,y,z)
            ac("✅ 已传送")
        end
end})
reg("spawn", {Rank="Helper", Category="Teleport", Desc="重生", Args="[玩家]",
    Func=function(_, a)
        local t = LP; if a[1] then t = findP(a[1]) or (ac("找不到"); return) end
        if t == LP then
            if LP.Character then LP.Character:BreakJoints() end
            task.wait(0.5); LP:LoadCharacter()
        else
            if t.Character then t.Character:BreakJoints() end
        end
        ac("✅ 已重生 "..t.Name)
end})
reg("tpall", {Rank="Admin", Category="Teleport", Desc="传送全部玩家到身边", Func=function()
    if not LP.Character or not LP.Character:FindFirstChild("HumanoidRootPart") then return end
    local cf = LP.Character.HumanoidRootPart.CFrame; local n = 0
    for _, p in pairs(Players:GetPlayers()) do
        if p~=LP and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
            p.Character.HumanoidRootPart.CFrame = cf + Vector3.new(math.random(-5,5), 2, math.random(-5,5))
            n = n+1
        end
    end
    ac("✅ TP了 "..n.." 人")
end})

-- ===== PLAYER =====
reg("kill", {Rank="Moderator", Category="Player", Desc="击杀玩家", Args="<玩家>",
    Func=function(_, a)
        if #a<1 then return ac("用法: ;kill <玩家>") end
        local t = findP(a[1])
        if t and t.Character and t.Character:FindFirstChild("Humanoid") then
            t.Character.Humanoid.Health = 0
            ac("💀 "..t.Name)
        end
end})
reg("killall", {Rank="Admin", Category="Player", Desc="击杀所有玩家", Func=function()
    local n = 0
    for _, p in pairs(Players:GetPlayers()) do
        if p~=LP and p.Character and p.Character:FindFirstChild("Humanoid") then
            p.Character.Humanoid.Health = 0; n = n+1
        end
    end
    ac("💀 击杀 "..n.." 人")
end})
reg("heal", {Rank="Helper", Category="Player", Desc="治疗玩家", Args="[玩家]",
    Func=function(_, a)
        local t = a[1] and findP(a[1]) or LP
        if t.Character and t.Character:FindFirstChild("Humanoid") then
            t.Character.Humanoid.Health = t.Character.Humanoid.MaxHealth
            ac("💚 已治疗 "..t.Name)
        end
end})
reg("healall", {Rank="Admin", Category="Player", Desc="治疗所有玩家", Func=function()
    local n = 0
    for _, p in pairs(Players:GetPlayers()) do
        if p.Character and p.Character:FindFirstChild("Humanoid") then
            p.Character.Humanoid.Health = p.Character.Humanoid.MaxHealth; n = n+1
        end
    end
    ac("💚 治疗 "..n.." 人")
end})
reg("freeze", {Rank="Moderator", Category="Player", Desc="冻结玩家", Args="<玩家>",
    Func=function(_, a)
        if #a<1 then return ac("用法: ;freeze <玩家>") end
        local t = findP(a[1])
        if t and t.Character then
            local r = t.Character:FindFirstChild("HumanoidRootPart") or t.Character:FindFirstChild("Torso")
            if r then r.Anchored = true; ac("🧊 冻结 "..t.Name) end
        end
end})
reg("unfreeze", {Rank="Moderator", Category="Player", Desc="解冻玩家", Args="<玩家>",
    Func=function(_, a)
        if #a<1 then return ac("用法: ;unfreeze <玩家>") end
        local t = findP(a[1])
        if t and t.Character then
            local r = t.Character:FindFirstChild("HumanoidRootPart") or t.Character:FindFirstChild("Torso")
            if r then r.Anchored = false; ac("✅ 解冻 "..t.Name) end
        end
end})
reg("freezeall", {Rank="Admin", Category="Player", Desc="冻结所有玩家", Func=function()
    local n = 0
    for _, p in pairs(Players:GetPlayers()) do
        if p~=LP and p.Character then
            local r = p.Character:FindFirstChild("HumanoidRootPart") or p.Character:FindFirstChild("Torso")
            if r then r.Anchored = true; n=n+1 end
        end
    end
    ac("🧊 冻结 "..n.." 人")
end})
reg("unfreezeall", {Rank="Admin", Category="Player", Desc="解冻所有玩家", Func=function()
    local n = 0
    for _, p in pairs(Players:GetPlayers()) do
        if p.Character then
            local r = p.Character:FindFirstChild("HumanoidRootPart") or p.Character:FindFirstChild("Torso")
            if r then r.Anchored = false; n=n+1 end
        end
    end
    ac("✅ 解冻 "..n.." 人")
end})
reg("invisible", {Rank="Admin", Category="Player", Desc="隐身",
    Func=function()
        if LP.Character then
            for _, p in ipairs(LP.Character:GetDescendants()) do
                if p:IsA("BasePart") then p.Transparency = 1 end
            end
            local h = LP.Character:FindFirstChild("Humanoid")
            if h then h.Name = "H" end
            ac("👻 隐身")
        end
end})
reg("visible", {Rank="Admin", Category="Player", Desc="取消隐身",
    Func=function()
        if LP.Character then
            for _, p in ipairs(LP.Character:GetDescendants()) do
                if p:IsA("BasePart") then p.Transparency = 0 end
            end
            local h = LP.Character:FindFirstChild("H")
            if h then h.Name = "Humanoid" end
            ac("✅ 可见")
        end
end})
reg("speed", {Rank="Admin", Category="Player", Desc="设置移动速度 1-500", Args="<数值> [玩家]",
    Func=function(_, a)
        if #a<1 then return ac("用法: ;speed <数值> [玩家]") end
        local s = math.clamp(tonumber(a[1]) or 16, 1, 500)
        local t = a[2] and findP(a[2]) or LP
        if t.Character and t.Character:FindFirstChild("Humanoid") then
            t.Character.Humanoid.WalkSpeed = s
            ac("⚡ "..t.Name.." 速度="..s)
        end
end})
reg("jump", {Rank="Admin", Category="Player", Desc="设置跳跃高度 1-500", Args="<数值> [玩家]",
    Func=function(_, a)
        if #a<1 then return ac("用法: ;jump <数值> [玩家]") end
        local j = math.clamp(tonumber(a[1]) or 50, 1, 500)
        local t = a[2] and findP(a[2]) or LP
        if t.Character and t.Character:FindFirstChild("Humanoid") then
            t.Character.Humanoid.JumpPower = j
            ac("⬆ "..t.Name.." 跳跃="..j)
        end
end})
reg("sethp", {Rank="Admin", Category="Player", Desc="设置血量", Args="<血量> <玩家>",
    Func=function(_, a)
        if #a<2 then return ac("用法: ;sethp <血量> <玩家>") end
        local hp = tonumber(a[1]); local t = findP(a[2])
        if not (t and hp) then return end
        if t.Character and t.Character:FindFirstChild("Humanoid") then
            t.Character.Humanoid.Health = math.clamp(hp, 0, 999999)
            ac("✅ "..t.Name.." HP="..hp)
        end
end})

-- ===== WORLD =====
reg("time", {Rank="Admin", Category="World", Desc="设置时间 0-24", Args="<时>",
    Func=function(_, a)
        if #a<1 then return ac("用法: ;time <0-24>") end
        LG:SetMinutesAfterMidnight(math.clamp(tonumber(a[1]) or 12, 0, 24) * 60)
        ac("🕐 时间已设")
end})
reg("gravity", {Rank="Admin", Category="World", Desc="设置重力", Args="<数值>",
    Func=function(_, a)
        if #a<1 then return ac("用法: ;gravity <数值>") end
        WS.Gravity = math.clamp(tonumber(a[1]) or 196.2, 1, 2000)
        ac("🌍 重力="..WS.Gravity)
end})
reg("fog", {Rank="Admin", Category="World", Desc="设置雾气", Args="<距离>",
    Func=function(_, a)
        if #a<1 then return ac("用法: ;fog <距离>") end
        LG.FogEnd = tonumber(a[1]) or 1000
        ac("✅ 雾气已设")
end})
reg("nofog", {Rank="Admin", Category="World", Desc="清除雾气", Func=function()
    LG.FogEnd = 100000; ac("✅ 雾已清除")
end})

-- ===== FUN =====
reg("explode", {Rank="Admin", Category="Fun", Desc="爆炸玩家", Args="<玩家>",
    Func=function(_, a)
        if #a<1 then return ac("用法: ;explode <玩家>") end
        local t = findP(a[1])
        if t and t.Character and t.Character:FindFirstChild("HumanoidRootPart") then
            local e = Instance.new("Explosion")
            e.Position = t.Character.HumanoidRootPart.Position
            e.BlastRadius = 12; e.BlastPressure = 8000; e.Parent = WS
            ac("💥 "..t.Name)
        end
end})
reg("explodeall", {Rank="Admin", Category="Fun", Desc="全部爆炸", Func=function()
    for _, p in pairs(Players:GetPlayers()) do
        if p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
            local e = Instance.new("Explosion")
            e.Position = p.Character.HumanoidRootPart.Position
            e.BlastRadius = 10; e.BlastPressure = 5000; e.Parent = WS
        end
    end
    ac("💥 BOOM!")
end})
reg("fire", {Rank="Admin", Category="Fun", Desc="点火玩家", Args="<玩家>",
    Func=function(_, a)
        if #a<1 then return ac("用法: ;fire <玩家>") end
        local t = findP(a[1])
        if t and t.Character then
            local p = t.Character:FindFirstChild("HumanoidRootPart") or t.Character:FindFirstChild("Torso") or t.Character
            local f = Instance.new("Fire"); f.Heat = 20; f.Size = 10; f.Parent = p; DS:AddItem(f,20)
            ac("🔥 "..t.Name)
        end
end})
reg("sparkles", {Rank="Helper", Category="Fun", Desc="闪粉效果", Args="[玩家]",
    Func=function(_, a)
        local t = a[1] and findP(a[1]) or LP
        if t.Character then
            local p = t.Character:FindFirstChild("HumanoidRootPart") or t.Character:FindFirstChild("Torso") or t.Character
            local s = Instance.new("Sparkles"); s.Parent = p; DS:AddItem(s,20)
            ac("✨")
        end
end})
reg("ff", {Rank="Admin", Category="Fun", Desc="护盾玩家", Args="<玩家>",
    Func=function(_, a)
        if #a<1 then return ac("用法: ;ff <玩家>") end
        local t = findP(a[1])
        if t and t.Character then
            local f = Instance.new("ForceField"); f.Parent = t.Character; DS:AddItem(f,30)
            ac("🛡 "..t.Name)
        end
end})
reg("ffall", {Rank="Admin", Category="Fun", Desc="全部护盾", Func=function()
    for _, p in pairs(Players:GetPlayers()) do
        if p.Character then
            local f = Instance.new("ForceField"); f.Parent = p.Character; DS:AddItem(f,30)
        end
    end
    ac("🛡 全员护盾")
end})
reg("throw", {Rank="Moderator", Category="Fun", Desc="扔飞玩家", Args="<玩家>",
    Func=function(_, a)
        if #a<1 then return ac("用法: ;throw <玩家>") end
        local t = findP(a[1])
        if t and t.Character and t.Character:FindFirstChild("HumanoidRootPart") then
            local bv = Instance.new("BodyVelocity")
            bv.Velocity = Vector3.new(math.random(-60,60), 80, math.random(-60,60))
            bv.MaxForce = Vector3.new(5000,5000,5000); bv.Parent = t.Character.HumanoidRootPart
            DS:AddItem(bv, 3); ac("🚀 "..t.Name)
        end
end})
reg("loopkill", {Rank="Admin", Category="Fun", Desc="循环击杀(20次)", Args="<玩家>",
    Func=function(_, a)
        if #a<1 then return ac("用法: ;loopkill <玩家>") end
        local t = findP(a[1]); if not t then return ac("找不到") end
        task.spawn(function()
            for i=1,20 do
                if not t or not t.Parent then break end
                if t.Character and t.Character:FindFirstChild("Humanoid") then t.Character.Humanoid.Health = 0 end
                task.wait(2)
            end
        end)
        ac("🔄 开始循环击杀 "..t.Name)
end})
reg("loopheal", {Rank="Admin", Category="Fun", Desc="持续治疗(10次)", Args="[玩家]",
    Func=function(_, a)
        local t = a[1] and findP(a[1]) or LP
        task.spawn(function()
            for i=1,10 do
                if not t or not t.Parent then break end
                if t.Character and t.Character:FindFirstChild("Humanoid") then
                    t.Character.Humanoid.Health = t.Character.Humanoid.MaxHealth
                end
                task.wait(1)
            end
        end)
        ac("💚 开始治疗 "..t.Name)
end})
reg("tpon", {Rank="Admin", Category="Fun", Desc="追踪传送玩家(5秒)", Args="<玩家>",
    Func=function(_, a)
        if #a<1 then return ac("用法: ;tpon <玩家>") end
        local t = findP(a[1]); if not t then return ac("找不到") end
        task.spawn(function()
            for i=1,50 do
                if not t or not t.Parent then break end
                if t.Character and t.Character:FindFirstChild("HumanoidRootPart") and
                   LP.Character and LP.Character:FindFirstChild("HumanoidRootPart") then
                    LP.Character.HumanoidRootPart.CFrame = t.Character.HumanoidRootPart.CFrame + Vector3.new(0,3,0)
                end
                task.wait(0.1)
            end
        end)
        ac("📍 追踪 "..t.Name)
end})
reg("fly", {Rank="Admin", Category="Player", Desc="飞行模式", Args="<on/off>",
    Func=function(_, a)
        if #a<1 then return ac("用法: ;fly <on/off>") end
        local state = a[1]:lower()=="on"
        if state then
            local char = LP.Character; local hrp = char and char:FindFirstChild("HumanoidRootPart")
            if not hrp then return end
            local hum = char:FindFirstChildOfClass("Humanoid")
            if hum then hum.PlatformStand = true end
            local bv = Instance.new("BodyVelocity"); bv.Name = "flyBV"
            bv.Velocity = Vector3.zero; bv.MaxForce = Vector3.new(1e5,1e5,1e5); bv.Parent = hrp
            RunS.RenderStepped:Connect(function()
                if not state then return end
                local c2 = LP.Character; local r2 = c2 and c2:FindFirstChild("HumanoidRootPart")
                if not r2 then return end
                local cam = Camera; local md = Vector3.zero
                if UI:IsKeyDown(Enum.KeyCode.W) then md = md + cam.CFrame.LookVector end
                if UI:IsKeyDown(Enum.KeyCode.S) then md = md - cam.CFrame.LookVector end
                if UI:IsKeyDown(Enum.KeyCode.A) then md = md - cam.CFrame.RightVector end
                if UI:IsKeyDown(Enum.KeyCode.D) then md = md + cam.CFrame.RightVector end
                if UI:IsKeyDown(Enum.KeyCode.Space) then md = md + Vector3.new(0,1,0) end
                if UI:IsKeyDown(Enum.KeyCode.LeftShift) then md = md - Vector3.new(0,1,0) end
                local b = r2:FindFirstChild("flyBV")
                if b then b.Velocity = md.Magnitude>0 and md.Unit*50 or Vector3.zero end
            end)
            ac("✈️ 飞行 ON")
        else
            local char = LP.Character; local hum = char and char:FindFirstChildOfClass("Humanoid")
            if hum then hum.PlatformStand = false end
            local hrp = char and char:FindFirstChild("HumanoidRootPart")
            if hrp then local b = hrp:FindFirstChild("flyBV"); if b then b:Destroy() end end
            ac("飞行 OFF")
        end
end})
reg("esp", {Rank="Admin", Category="Utility", Desc="ESP透视", Args="<on/off>",
    Func=function(_, a)
        if #a<1 then return ac("用法: ;esp <on/off>") end
        local state = a[1]:lower()=="on"
        if state then
            for _, p in pairs(Players:GetPlayers()) do
                if p~=LP then
                    local h = Instance.new("Highlight"); h.Name = "AdminESP"
                    h.FillColor = Color3.fromRGB(255,50,50); h.OutlineColor = Color3.new(1,1,1)
                    h.FillTransparency = 0.4
                    h.Adornee = p.Character or p.CharacterAdded:Wait()
                    h.Parent = LP:WaitForChild("PlayerGui")
                end
            end
        else
            for _, v in pairs(LP:WaitForChild("PlayerGui"):GetChildren()) do
                if v.Name == "AdminESP" then v:Destroy() end
            end
        end
        ac("ESP: "..(state and "ON" or "OFF"))
end})
reg("exec", {Rank="Owner", Category="Management", Desc="执行Lua代码", Args="<代码>",
    Func=function(_, a)
        if #a<1 then return end
        local fn, err = loadstring(table.concat(a, " "))
        if not fn then return ac("❌ "..err) end
        local ok, res = pcall(fn)
        ac(ok and "✅" or "❌ "..tostring(res))
end})

-- ===== CHAT HOOK =====
local function hookChat()
    local chatRemote = nil
    for _, o in pairs(RS:GetDescendants()) do
        if o:IsA("RemoteEvent") and (o.Name:find("Chat") or o.Name:find("Say") or o.Name:find("Message") or o.Name:find("Talk")) then
            chatRemote = o; break
        end
    end
    if chatRemote and hookfunction and newcclosure then
        local old
        old = hookfunction(chatRemote.FireServer, newcclosure(function(self, ...)
            local msg = tostring(select(1, ...) or "")
            if msg:sub(1,1) == PREFIX then
                processCmd(msg)
                return
            end
            return old(self, ...)
        end))
        ac("✅ 聊天钩子已安装")
    else
        ac("ℹ️ hookfunction 不可用，请用面板输入框发指令")
    end
    pcall(function()
        for _, v in pairs(getconnections(LP.Chatted)) do v:Disable() end
    end)
end

-- ===== COMMAND PROCESSOR =====
local function processCmd(msg)
    if msg:sub(1,1) ~= PREFIX then return end
    local now = tick()
    if Cooldowns[LP] and now - Cooldowns[LP] < 0.3 then return end
    Cooldowns[LP] = now
    local parts = {}; for w in msg:sub(2):gmatch("%S+") do parts[#parts+1] = w end
    if #parts == 0 then return end
    local cmd = Cmds[parts[1]:lower()]
    if not cmd then ac("❌ 未知指令: "..parts[1].." 用 "..PREFIX.."cmds"); return end
    if not hasPerm(LP, cmd.Rank) then ac("❌ 无权限"); return end
    local args = {}; for i=2,#parts do args[#args+1] = parts[i] end
    local ok, err = pcall(cmd.Func, LP, args)
    if not ok then ac("❌ 错误: "..tostring(err)) end
end

-- ===== UI =====
local PlayerGui = LP:WaitForChild("PlayerGui")
local vp = Camera.ViewportSize

local sg = Instance.new("ScreenGui")
sg.Name = "AdminPanel"; sg.ResetOnSpawn = false; sg.Parent = PlayerGui

local fw = math.min(440, vp.X - 20)
local fh = math.min(360, vp.Y - 20)

local main = Instance.new("Frame")
main.Size = UDim2.new(0, fw, 0, fh)
main.Position = UDim2.new(0.5, -fw/2, 0.5, -fh/2)
main.BackgroundColor3 = Color3.fromRGB(8, 0, 20)
main.BorderSizePixel = 0; main.Active = true; main.Parent = sg
Instance.new("UICorner", main).CornerRadius = UDim.new(0, 10)
local st = Instance.new("UIStroke", main)
st.Color = Color3.fromRGB(120, 0, 255); st.Thickness = 2

-- Title bar
local titleBar = Instance.new("TextLabel")
titleBar.Size = UDim2.new(1, -50, 0, 30)
titleBar.Position = UDim2.new(0, 8, 0, 5)
titleBar.Text = "🚀 Universal Admin  |  "..getRank(LP).." (Lv."..getLv(LP)..")"
titleBar.Font = Enum.Font.GothamBold; titleBar.TextSize = 13
titleBar.TextColor3 = Color3.fromRGB(120, 0, 255)
titleBar.BackgroundTransparency = 1; titleBar.TextXAlignment = Enum.TextXAlignment.Left
titleBar.Parent = main

-- Drag
local drag, ds, dp
titleBar.InputBegan:Connect(function(i)
    if i.UserInputType==Enum.UserInputType.MouseButton1 or i.UserInputType==Enum.UserInputType.Touch then
        drag=true; ds=i.Position; dp=main.Position
        i.Changed:Connect(function() if i.UserInputState==Enum.UserInputState.End then drag=false end end)
    end
end)
titleBar.InputChanged:Connect(function(i)
    if drag and (i.UserInputType==Enum.UserInputType.MouseMovement or i.UserInputType==Enum.UserInputType.Touch) then
        local d = i.Position-ds
        main.Position = UDim2.new(dp.X.Scale, dp.X.Offset+d.X, dp.Y.Scale, dp.Y.Offset+d.Y)
    end
end)

-- Close
local closeBtn = Instance.new("TextButton")
closeBtn.Size = UDim2.new(0, 22, 0, 22); closeBtn.Position = UDim2.new(1, -28, 0, 9)
closeBtn.Text = "✕"; closeBtn.Font = Enum.Font.GothamBold; closeBtn.TextSize = 12
closeBtn.TextColor3 = Color3.new(1,1,1)
closeBtn.BackgroundColor3 = Color3.fromRGB(200,50,50); closeBtn.BorderSizePixel = 0
Instance.new("UICorner", closeBtn).CornerRadius = UDim.new(0, 5); closeBtn.Parent = main
closeBtn.MouseButton1Click:Connect(function() sg:Destroy() end)

-- Tabs
local tabs = {"📋指令","👤玩家","⚡工具","🎮娱乐","⚙设置"}
local tabBtns = {}; local tabPages = {}
local tabColors = {Color3.fromRGB(120,0,255), Color3.fromRGB(0,150,255),
                   Color3.fromRGB(255,140,0), Color3.fromRGB(255,80,80), Color3.fromRGB(100,200,100)}

for i, tn in ipairs(tabs) do
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0.2, -2, 0, 26); btn.Position = UDim2.new((i-1)*0.2, 1, 0, 38)
    btn.Text = tn; btn.Font = Enum.Font.GothamBold; btn.TextSize = 10; btn.TextColor3 = Color3.new(1,1,1)
    btn.BackgroundColor3 = i==1 and tabColors[1] or Color3.fromRGB(30,10,50)
    btn.BorderSizePixel = 0
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 5); btn.Parent = main
    tabBtns[i] = btn

    -- Page content
    local page = Instance.new("ScrollingFrame")
    page.Size = UDim2.new(1, -12, 0, fh-110)
    page.Position = UDim2.new(0, 6, 0, 68)
    page.BackgroundTransparency = 1; page.BorderSizePixel = 0
    page.ScrollBarThickness = 3; page.ScrollBarImageColor3 = Color3.fromRGB(120,0,255)
    page.CanvasSize = UDim2.new(0,0,0,0); page.Visible = i==1; page.Parent = main
    local layout = Instance.new("UIListLayout", page); layout.Padding = UDim.new(0, 2)
    tabPages[i] = page

    btn.MouseButton1Click:Connect(function()
        for j=1,#tabs do
            tabBtns[j].BackgroundColor3 = j==i and tabColors[j] or Color3.fromRGB(30,10,50)
            tabPages[j].Visible = j==i
        end
        if i==2 then refreshPlayerList() end
    end)
end

-- Tab 1: Commands
local cmdCount = 0; local seen = {}
for n,c in pairs(Cmds) do
    if not seen[n] then
        seen[n] = true
        if hasPerm(LP, c.Rank) then
            cmdCount = cmdCount + 1
            local l = Instance.new("TextLabel")
            l.Size = UDim2.new(1, -6, 0, 16)
            l.Text = PREFIX..n.." "..c.Args.."  → "..c.Desc
            l.Font = Enum.Font.Gotham; l.TextSize = 10
            l.TextColor3 = Color3.fromRGB(210,200,220)
            l.BackgroundTransparency = 1; l.TextXAlignment = Enum.TextXAlignment.Left
            l.Parent = tabPages[1]
        end
    end
end
tabPages[1].CanvasSize = UDim2.new(0,0,0, cmdCount*18)

-- Tab 2: Players
local function refreshPlayerList()
    for _, c in pairs(tabPages[2]:GetChildren()) do
        if c:IsA("Frame") then c:Destroy() end
    end
    for _, p in pairs(Players:GetPlayers()) do
        local row = Instance.new("Frame")
        row.Size = UDim2.new(1, -4, 0, 30)
        row.BackgroundColor3 = Color3.fromRGB(20,8,35); row.BorderSizePixel = 0
        Instance.new("UICorner", row).CornerRadius = UDim.new(0, 5); row.Parent = tabPages[2]

        local nl = Instance.new("TextLabel")
        nl.Size = UDim2.new(0.35, -6, 1, 0); nl.Position = UDim2.new(0, 6, 0, 0)
        nl.Text = p.Name; nl.Font = Enum.Font.GothamBold; nl.TextSize = 10
        nl.TextColor3 = Color3.fromRGB(220,210,240); nl.BackgroundTransparency = 1
        nl.TextXAlignment = Enum.TextXAlignment.Left; nl.Parent = row

        local rl = Instance.new("TextLabel")
        rl.Size = UDim2.new(0, 40, 1, 0); rl.Position = UDim2.new(0.35, 0, 0, 0)
        rl.Text = getRank(p):sub(1,4); rl.Font = Enum.Font.Gotham; rl.TextSize = 8
        rl.TextColor3 = Color3.fromRGB(150,140,175); rl.BackgroundTransparency = 1; rl.Parent = row

        local btns = {{"TP",";tp ",Color3.fromRGB(80,0,160)},{"K",";kill ",Color3.fromRGB(255,60,60)},
                      {"H",";heal ",Color3.fromRGB(50,220,80)},{"F",";freeze ",Color3.fromRGB(100,100,255)}}
        for j,b in ipairs(btns) do
            local btn = Instance.new("TextButton")
            btn.Size = UDim2.new(0, 22, 0, 20); btn.Position = UDim2.new(0.6+(j-1)*0.07, 0, 0.5, -10)
            btn.Text = b[1]; btn.Font = Enum.Font.GothamBold; btn.TextSize = 8; btn.TextColor3 = Color3.new(1,1,1)
            btn.BackgroundColor3 = b[3]; btn.BorderSizePixel = 0
            Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 4); btn.Parent = row
            local cmd = b[2]..p.Name
            btn.MouseButton1Click:Connect(function() processCmd(cmd) end)
        end
    end
    tabPages[2].CanvasSize = UDim2.new(0,0,0, 32*#Players:GetPlayers()+10)
end
refreshPlayerList()
Players.PlayerAdded:Connect(refreshPlayerList)
Players.PlayerRemoving:Connect(refreshPlayerList)

-- Tab 3: Quick Tools
local toolBtns = {
    {"Kill All",";killall"}, {"Heal All",";healall"},
    {"Freeze All",";freezeall"}, {"Unfreeze All",";unfreezeall"},
    {"TP All",";tpall"}, {"ESP ON",";esp on"},
    {"ESP OFF",";esp off"}, {"Fly",";fly on"},
    {"Stop Fly",";fly off"}, {"Invisible",";invisible"},
    {"Visible",";visible"}, {"FF All",";ffall"},
}
local tg = Instance.new("UIGridLayout", tabPages[3])
tg.CellSize = UDim2.new(0.5, -4, 0, 28); tg.CellPadding = UDim2.new(0, 4, 0, 4)
for _, tb in ipairs(toolBtns) do
    local btn = Instance.new("TextButton")
    btn.Text = tb[1]; btn.Font = Enum.Font.GothamBold; btn.TextSize = 10; btn.TextColor3 = Color3.new(1,1,1)
    btn.BackgroundColor3 = Color3.fromRGB(60, 20, 100); btn.BorderSizePixel = 0
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 5); btn.Parent = tabPages[3]
    local cmd = tb[2]
    btn.MouseButton1Click:Connect(function() processCmd(cmd) end)
end

-- Tab 4: Fun
local funBtns = {
    {"Explode All",";explodeall"}, {"FF All",";ffall"},
    {"Sparkles",";sparkles"}, {"Throw All",";throw all"},
    {"Loop Kill",";loopkill "}, {"BOOM!",";explodeall"},
}
local fg = Instance.new("UIGridLayout", tabPages[4])
fg.CellSize = UDim2.new(0.5, -4, 0, 28); fg.CellPadding = UDim2.new(0, 4, 0, 4)
for _, fb in ipairs(funBtns) do
    local btn = Instance.new("TextButton")
    btn.Text = fb[1]; btn.Font = Enum.Font.GothamBold; btn.TextSize = 10; btn.TextColor3 = Color3.new(1,1,1)
    btn.BackgroundColor3 = Color3.fromRGB(120, 30, 30); btn.BorderSizePixel = 0
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 5); btn.Parent = tabPages[4]
    local cmd = fb[2]
    btn.MouseButton1Click:Connect(function() processCmd(cmd) end)
end

-- Tab 5: Settings
local setItems = {
    {"你的权限等级", getRank(LP).." (Lv."..getLv(LP)..")"},
    {"检查更新", ";cmds 查看指令"},
    {"指令前缀", PREFIX},
}
for _, si in ipairs(setItems) do
    local row = Instance.new("Frame")
    row.Size = UDim2.new(1, -4, 0, 26)
    row.BackgroundColor3 = Color3.fromRGB(20,8,35); row.BorderSizePixel = 0
    Instance.new("UICorner", row).CornerRadius = UDim.new(0, 5); row.Parent = tabPages[5]
    local tl = Instance.new("TextLabel")
    tl.Size = UDim2.new(0.4, -6, 1, 0); tl.Position = UDim2.new(0, 6, 0, 0)
    tl.Text = si[1]; tl.Font = Enum.Font.Gotham; tl.TextSize = 10; tl.TextColor3 = Color3.fromRGB(200,200,200)
    tl.BackgroundTransparency = 1; tl.TextXAlignment = Enum.TextXAlignment.Left; tl.Parent = row
    local vl = Instance.new("TextLabel")
    vl.Size = UDim2.new(0.55, -6, 1, 0); vl.Position = UDim2.new(0.4, 6, 0, 0)
    vl.Text = si[2]; vl.Font = Enum.Font.GothamBold; vl.TextSize = 10; vl.TextColor3 = Color3.fromRGB(120,0,255)
    vl.BackgroundTransparency = 1; vl.TextXAlignment = Enum.TextXAlignment.Right; vl.Parent = row
end

-- Command bar
local cmdBar = Instance.new("Frame")
cmdBar.Size = UDim2.new(1, -14, 0, 30); cmdBar.Position = UDim2.new(0, 7, 1, -74)
cmdBar.BackgroundColor3 = Color3.fromRGB(20, 5, 40); cmdBar.BorderSizePixel = 0
Instance.new("UICorner", cmdBar).CornerRadius = UDim.new(0, 6); cmdBar.Parent = main

local inputBox = Instance.new("TextBox")
inputBox.Size = UDim2.new(1, -10, 1, 0); inputBox.Position = UDim2.new(0, 6, 0, 0)
inputBox.PlaceholderText = "输入 "..PREFIX.."cmds 查看指令列表"
inputBox.Text = ""; inputBox.Font = Enum.Font.Gotham; inputBox.TextSize = 11
inputBox.TextColor3 = Color3.fromRGB(220,210,240)
inputBox.PlaceholderColor3 = Color3.fromRGB(80,70,100)
inputBox.BackgroundTransparency = 1; inputBox.ClearTextOnFocus = false; inputBox.Parent = cmdBar

-- Status bar
local statusBar = Instance.new("TextLabel")
statusBar.Size = UDim2.new(1, -14, 0, 30); statusBar.Position = UDim2.new(0, 7, 1, -38)
statusBar.Text = cmdCount.." commands  |  权限: "..getRank(LP).."  |  RightShift = 开关面板"
statusBar.Font = Enum.Font.Gotham; statusBar.TextSize = 9
statusBar.TextColor3 = Color3.fromRGB(100,90,120)
statusBar.BackgroundTransparency = 1; statusBar.TextXAlignment = Enum.TextXAlignment.Left; statusBar.Parent = main

-- Send command
local function sendCmd()
    local txt = inputBox.Text:match("^%s*(.-)%s*$")
    if txt == "" then return end
    if txt:sub(1,1) ~= PREFIX then txt = PREFIX..txt end
    table.insert(CmdHistory, 1, txt); CmdIdx = 0
    processCmd(txt)
    inputBox.Text = ""
end
inputBox.FocusLost:Connect(function(enter) if enter then sendCmd() end end)

-- Hotkeys
UI.InputBegan:Connect(function(i, g)
    if g then return end
    if i.KeyCode == Enum.KeyCode.RightShift then main.Visible = not main.Visible end
    if i.KeyCode == Enum.KeyCode.Escape then main.Visible = false end
    if i.KeyCode == Enum.KeyCode.Up and inputBox:IsFocused() and #CmdHistory>0 then
        CmdIdx = math.min(CmdIdx+1, #CmdHistory)
        inputBox.Text = CmdHistory[CmdIdx]:gsub("^"..PREFIX, "")
    end
    if i.KeyCode == Enum.KeyCode.Down and inputBox:IsFocused() and CmdIdx>0 then
        CmdIdx = CmdIdx - 1
        inputBox.Text = CmdIdx>0 and CmdHistory[CmdIdx]:gsub("^"..PREFIX, "") or ""
    end
end)

-- ===== INIT =====
hookChat()
ac("✅ 已加载! "..cmdCount.." 指令 | 权限: "..getRank(LP).." | "..PREFIX.."cmds")
nfy("Admin Panel", "✅ "..cmdCount.." commands | 权限: "..getRank(LP), 4)

-- Auto-save ranks
task.spawn(function() while task.wait(120) do saveData() end end)
