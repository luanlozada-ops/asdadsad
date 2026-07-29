-- UBG STANDALONE — Kill Mob + Lag Server V3
-- Tecla 5: toggle ON/OFF ambos
-- UI: notificacion simple en pantalla

local Players           = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService        = game:GetService("RunService")
local UserInputService  = game:GetService("UserInputService")
local LocalPlayer       = Players.LocalPlayer

-- =====================================================
-- CACHE DE REMOTES
-- =====================================================
local Remotes = {
    Action          = nil,
    Ability         = nil,
    AbilityCanceled = nil,
    ChangeCharacter = nil,
    MobAbilities    = {},
    ready           = false,
}

task.spawn(function()
    pcall(function()
        local r = ReplicatedStorage:WaitForChild("Remotes", 10)
        Remotes.Action          = r:WaitForChild("Combat"):WaitForChild("Action", 5)
        Remotes.Ability         = r:WaitForChild("Abilities"):WaitForChild("Ability", 5)
        Remotes.AbilityCanceled = r:WaitForChild("Abilities"):WaitForChild("AbilityCanceled", 5)
        Remotes.ChangeCharacter = r:WaitForChild("Character"):WaitForChild("ChangeCharacter", 5)
        local mob = ReplicatedStorage:WaitForChild("Characters"):WaitForChild("Mob", 5)
        if mob then
            local ab = mob:FindFirstChild("Abilities")
            if ab then
                for i = 1, 4 do
                    Remotes.MobAbilities[i] = ab:FindFirstChild(tostring(i))
                end
            end
        end
        Remotes.ready = true
    end)
end)

-- =====================================================
-- UI SIMPLE (BillboardGui en ScreenGui)
-- =====================================================
local ui = Instance.new("ScreenGui")
ui.Name = "StandaloneUI"
ui.ResetOnSpawn = false
ui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
pcall(function() ui.Parent = game:GetService("CoreGui") end)
if not ui.Parent then ui.Parent = LocalPlayer.PlayerGui end

local frame = Instance.new("Frame")
frame.Size = UDim2.new(0, 200, 0, 60)
frame.Position = UDim2.new(0.5, -100, 0, 10)
frame.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
frame.BackgroundTransparency = 0.3
frame.BorderSizePixel = 0
frame.Parent = ui

local corner = Instance.new("UICorner")
corner.CornerRadius = UDim.new(0, 6)
corner.Parent = frame

frame.Size = UDim2.new(0, 220, 0, 80)

local label = Instance.new("TextLabel")
label.Size = UDim2.new(1, 0, 0.5, 0)
label.Position = UDim2.new(0, 0, 0, 0)
label.BackgroundTransparency = 1
label.TextColor3 = Color3.fromRGB(255, 255, 255)
label.TextSize = 13
label.Font = Enum.Font.GothamBold
label.Text = "⬛ [5] Kill Mob+Lag - OFF"
label.Parent = frame

local labelFarm = Instance.new("TextLabel")
labelFarm.Size = UDim2.new(1, 0, 0.5, 0)
labelFarm.Position = UDim2.new(0, 0, 0.5, 0)
labelFarm.BackgroundTransparency = 1
labelFarm.TextColor3 = Color3.fromRGB(255, 255, 255)
labelFarm.TextSize = 13
labelFarm.Font = Enum.Font.GothamBold
labelFarm.Text = "⬛ [6] Auto Farm - OFF"
labelFarm.Parent = frame

local function setStatus(on)
    if on then
        label.Text = "🟢 [5] Kill Mob+Lag - ON"
        label.TextColor3 = Color3.fromRGB(100, 255, 100)
    else
        label.Text = "⬛ [5] Kill Mob+Lag - OFF"
        label.TextColor3 = Color3.fromRGB(255, 255, 255)
    end
end

local function setFarmStatus(on)
    if on then
        labelFarm.Text = "🟢 [6] Auto Farm - ON"
        labelFarm.TextColor3 = Color3.fromRGB(100, 255, 100)
    else
        labelFarm.Text = "⬛ [6] Auto Farm - OFF"
        labelFarm.TextColor3 = Color3.fromRGB(255, 255, 255)
    end
end


-- =====================================================
-- MASS KILL (para AutoFarm)
-- =====================================================
local function MassKill(targets, stackCount)
    if not targets or #targets == 0 then return end
    local ok, charVal = pcall(function() return LocalPlayer.Data.Character.Value end)
    if not ok or not charVal then return end
    local charFolder = ReplicatedStorage:FindFirstChild("Characters") and ReplicatedStorage.Characters:FindFirstChild(charVal)
    if not charFolder then return end
    local wallCombo = charFolder:FindFirstChild("WallCombo")
    if not wallCombo then return end
    stackCount = stackCount or 50
    if charVal == "Gon" then stackCount = 20 end
    local multiHitList = {}
    for _, victimChar in ipairs(targets) do
        for i = 1, stackCount do
            table.insert(multiHitList, victimChar)
        end
    end
    pcall(function()
        Remotes.Ability:FireServer(wallCombo, 69)
        Remotes.Action:FireServer(wallCombo, "", 4, 69, {
            BestHitCharacter = nil,
            HitCharacters    = multiHitList,
            Ignore           = {},
            Actions          = {}
        })
    end)
end

-- =====================================================
-- AUTO FARM
-- =====================================================
local AutoFarm = {
    enabled    = false,
    wcConn     = nil,
    claimThread = nil,
}

local function afGetAlive()
    local alive = {}
    for _, p in ipairs(Players:GetPlayers()) do
        if p == LocalPlayer then continue end
        if not p.Character then continue end
        local hum = p.Character:FindFirstChildOfClass("Humanoid")
        local hp  = hum and (hum:GetAttribute("Health") or hum.Health) or 0
        if hp > 0 then table.insert(alive, p) end
    end
    return alive
end

local function startAutoFarm()
    if AutoFarm.wcConn then return end
    AutoFarm.enabled = true

    local afQueue    = {}
    local afQueueIdx = 1
    local afFrameCount = 0
    local AF_FRAMES_PER_TARGET = 3

    AutoFarm.wcConn = RunService.RenderStepped:Connect(function()
        if not AutoFarm.enabled then return end
        local myChar = LocalPlayer.Character
        local myHRP  = myChar and myChar:FindFirstChild("HumanoidRootPart")
        if not myHRP then return end

        if afQueueIdx > #afQueue then
            afQueue    = afGetAlive()
            afQueueIdx = 1
            afFrameCount = 0
            if #afQueue == 0 then return end
        end

        local p = afQueue[afQueueIdx]
        if p and p.Character then
            local tHRP = p.Character:FindFirstChild("HumanoidRootPart")
            if tHRP then
                myHRP.CFrame = CFrame.new(
                    tHRP.Position.X,
                    tHRP.Position.Y - 10,
                    tHRP.Position.Z
                )
                MassKill({p.Character}, 50)
            end
        end

        afFrameCount = afFrameCount + 1
        if afFrameCount >= AF_FRAMES_PER_TARGET then
            afFrameCount = 0
            afQueueIdx   = afQueueIdx + 1
        end
    end)

    AutoFarm.claimThread = task.spawn(function()
        while AutoFarm.enabled do
            pcall(function()
                ReplicatedStorage.Remotes.Combat.EmoteClaim:FireServer()
            end)
            task.wait(0.5)
        end
    end)
end

local function stopAutoFarm()
    AutoFarm.enabled = false
    if AutoFarm.wcConn then
        AutoFarm.wcConn:Disconnect()
        AutoFarm.wcConn = nil
    end
    AutoFarm.claimThread = nil
    -- Restaurar camara
    pcall(function()
        local myChar = LocalPlayer.Character
        local hum = myChar and myChar:FindFirstChildOfClass("Humanoid")
        if hum then workspace.CurrentCamera.CameraSubject = hum end
    end)
end

-- =====================================================
-- KILL MOB
-- =====================================================
local KillMob = { enabled = false, thread = nil }

local function getTargets()
    local char = LocalPlayer.Character
    local hrp  = char and char:FindFirstChild("HumanoidRootPart")
    if not hrp then return {} end
    local result = {}
    for _, p in ipairs(Players:GetPlayers()) do
        if p == LocalPlayer then continue end
        if p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
            table.insert(result, p.Character)
        end
    end
    pcall(function()
        for _, npc in ipairs(workspace.Characters.NPCs:GetChildren()) do
            if npc:IsA("Model") and npc:FindFirstChild("HumanoidRootPart") then
                table.insert(result, npc)
            end
        end
    end)
    return result
end

local function spamAbility4(targets)
    local ability = Remotes.MobAbilities[4]
    if not ability then return end
    local actions = {377,380,383,384,385,387,389}
    for _, targetChar in ipairs(targets) do
        local hrp = targetChar:FindFirstChild("HumanoidRootPart")
        if not hrp then continue end
        local targetCF = hrp.CFrame
        local tPlayer  = Players:GetPlayerFromCharacter(targetChar)
        local tName    = tPlayer and tPlayer.Name or targetChar.Name
        pcall(function()
            Remotes.AbilityCanceled:FireServer(ability)
            Remotes.Ability:FireServer(ability, 9000000)
            for i = 1, 7 do
                local data = {
                    HitboxCFrames     = {targetCF, targetCF},
                    BestHitCharacter  = targetChar,
                    HitCharacters     = {targetChar},
                    Ignore            = i > 2 and {ActionNumber1 = {targetChar}} or {},
                    DeathInfo         = {},
                    BlockedCharacters = {},
                    HitInfo           = {IsFacing = not(i==1 or i==2), IsInFront = i<=2},
                    ServerTime        = tick(),
                    Actions           = i > 2 and {ActionNumber1 = {}} or {},
                    FromCFrame        = targetCF,
                }
                if i == 7 then
                    data.RockCFrame = targetCF
                    data.Actions = {ActionNumber1 = {[tName] = {
                        StartCFrameStr     = tostring(targetCF.X)..","..tostring(targetCF.Y)..","..tostring(targetCF.Z)..",0,0,0,0,0,0,0,0,0",
                        ImpulseVelocity    = Vector3.new(1901,-25000,291),
                        AbilityName        = "4",
                        RotVelocityStr     = "0,0,0",
                        VelocityStr        = "1.9,0.01,0.29",
                        Duration           = 2,
                        RotImpulseVelocity = Vector3.new(5868,-6649,-7414),
                        Seed               = math.random(1,1e6),
                        LookVectorStr      = "0.99,0,0.15"
                    }}}
                end
                Remotes.Action:FireServer(ability, "Mob:Abilities:4", i, 9000000, data, "Action"..actions[i], i==2 and 0.01 or nil)
                if i == 3 or i == 6 then task.wait() end
            end
        end)
    end
end

local function startKillMob()
    if KillMob.thread then return end
    pcall(function()
        if LocalPlayer.Data.Character.Value ~= "Mob" then
            Remotes.ChangeCharacter:FireServer("Mob")
            task.wait(1)
        end
    end)
    KillMob.thread = task.spawn(function()
        while KillMob.enabled do
            pcall(function() spamAbility4(getTargets()) end)
            task.wait(0.05)
        end
    end)
end

local function stopKillMob()
    KillMob.enabled = false
    KillMob.thread = nil
end

-- =====================================================
-- LAG SERVER V3
-- =====================================================
local LagServer = { enabled = false, threads = {} }

local function lagSpamAbility(abilityNum, targets)
    if not Remotes.ready then return end
    local ability = Remotes.MobAbilities[abilityNum]
    if not ability then return end
    local actions = {377,380,383,384,385,387,389}
    for _, targetChar in ipairs(targets) do
        local hrp = targetChar:FindFirstChild("HumanoidRootPart")
        if not hrp then continue end
        local targetCF = hrp.CFrame
        local tPlayer  = Players:GetPlayerFromCharacter(targetChar)
        local tName    = tPlayer and tPlayer.Name or targetChar.Name
        pcall(function()
            Remotes.AbilityCanceled:FireServer(ability)
            Remotes.Ability:FireServer(ability, 9000000)
            for i = 1, 7 do
                local data = {
                    HitboxCFrames     = {targetCF, targetCF},
                    BestHitCharacter  = targetChar,
                    HitCharacters     = {targetChar},
                    Ignore            = i > 2 and {ActionNumber1 = {targetChar}} or {},
                    DeathInfo         = {},
                    BlockedCharacters = {},
                    HitInfo           = {IsFacing = not(i==1 or i==2), IsInFront = i<=2},
                    ServerTime        = tick(),
                    Actions           = i > 2 and {ActionNumber1 = {}} or {},
                    FromCFrame        = targetCF,
                }
                if i == 7 then
                    data.RockCFrame = targetCF
                    data.Actions = {ActionNumber1 = {[tName] = {
                        StartCFrameStr     = tostring(targetCF.X)..","..tostring(targetCF.Y)..","..tostring(targetCF.Z)..",0,0,0,0,0,0,0,0,0",
                        ImpulseVelocity    = Vector3.new(1901,-25000,291),
                        AbilityName        = tostring(abilityNum),
                        RotVelocityStr     = "0,0,0",
                        VelocityStr        = "1.9,0.01,0.29",
                        Duration           = 2,
                        RotImpulseVelocity = Vector3.new(5868,-6649,-7414),
                        Seed               = math.random(1,1e6),
                        LookVectorStr      = "0.99,0,0.15"
                    }}}
                end
                Remotes.Action:FireServer(ability, "Mob:Abilities:"..abilityNum, i, 9000000, data, "Action"..actions[i], i==2 and 0.01 or nil)
                if i == 3 or i == 6 then task.wait() end
            end
        end)
    end
end

local function startLagServer()
    if LagServer.enabled and #LagServer.threads > 0 then return end
    LagServer.threads = {}
    LagServer.enabled = true
    task.spawn(function()
        local waited = 0
        while not Remotes.ready and waited < 10 do
            task.wait(0.5); waited = waited + 0.5
        end
        if not Remotes.ready then LagServer.enabled = false; return end
        pcall(function()
            if LocalPlayer.Data.Character.Value ~= "Mob" then
                Remotes.ChangeCharacter:FireServer("Mob")
                task.wait(1)
            end
        end)

        -- Thread 1: Ability 4 A
        table.insert(LagServer.threads, task.spawn(function()
            while LagServer.enabled do
                pcall(function() lagSpamAbility(4, getTargets()) end)
                task.wait()
            end
        end))
        -- Thread 2: Ability 4 B
        table.insert(LagServer.threads, task.spawn(function()
            while LagServer.enabled do
                pcall(function() lagSpamAbility(4, getTargets()) end)
                task.wait()
            end
        end))
        -- Thread 3: Ability 3 A
        table.insert(LagServer.threads, task.spawn(function()
            while LagServer.enabled do
                pcall(function() lagSpamAbility(3, getTargets()) end)
                task.wait()
            end
        end))
        -- Thread 4: Ability 3 B
        table.insert(LagServer.threads, task.spawn(function()
            while LagServer.enabled do
                pcall(function() lagSpamAbility(3, getTargets()) end)
                task.wait()
            end
        end))
        -- Thread 5: Ability 1/2 alternando
        table.insert(LagServer.threads, task.spawn(function()
            local toggle = true
            while LagServer.enabled do
                pcall(function() lagSpamAbility(toggle and 1 or 2, getTargets()) end)
                toggle = not toggle
                task.wait(0.02)
            end
        end))
        -- Thread 6: AbilityCanceled spam
        table.insert(LagServer.threads, task.spawn(function()
            while LagServer.enabled do
                pcall(function()
                    for i = 1, 4 do
                        local ab = Remotes.MobAbilities[i]
                        if ab then Remotes.AbilityCanceled:FireServer(ab) end
                    end
                end)
                task.wait(0.02)
            end
        end))
        -- Thread 7: WallCombo
        table.insert(LagServer.threads, task.spawn(function()
            while LagServer.enabled do
                pcall(function()
                    local mob = ReplicatedStorage.Characters:FindFirstChild("Mob")
                    local wc  = mob and mob:FindFirstChild("WallCombo")
                    if not wc then return end
                    for _, tc in ipairs(getTargets()) do
                        local hrp = tc:FindFirstChild("HumanoidRootPart")
                        if not hrp then continue end
                        Remotes.Ability:FireServer(wc, 9000000, nil, tc, hrp.Position)
                        Remotes.Action:FireServer(wc, "Mob:WallCombo", 1, 9000000, {
                            HitboxCFrames={}, BestHitCharacter=tc,
                            HitCharacters={tc}, Ignore={}, DeathInfo={},
                            Actions={}, HitInfo={IsFacing=true, IsInFront=true},
                            BlockedCharacters={}, FromCFrame=hrp.CFrame
                        }, "Action651", 0)
                    end
                end)
                task.wait()
            end
        end))
        -- Thread 8: Ability FireServer directo
        table.insert(LagServer.threads, task.spawn(function()
            while LagServer.enabled do
                pcall(function()
                    for _, tc in ipairs(getTargets()) do
                        for i = 1, 4 do
                            local ab = Remotes.MobAbilities[i]
                            if ab then Remotes.Ability:FireServer(ab, 9000000) end
                        end
                    end
                end)
                task.wait()
            end
        end))
    end)
end

local function stopLagServer()
    LagServer.enabled = false
    -- No usamos task.cancel -- los threads se detienen solos con la flag enabled=false
    LagServer.threads = {}
end

-- =====================================================
-- TOGGLE PRINCIPAL (tecla 5)
-- =====================================================
local Active = false

local function toggleAll()
    Active = not Active
    if Active then
        KillMob.enabled = true
        startKillMob()
        startLagServer()
    else
        stopKillMob()
        stopLagServer()
    end
    setStatus(Active)
end

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.KeyCode == Enum.KeyCode.Five then
        toggleAll()
    elseif input.KeyCode == Enum.KeyCode.Six then
        AutoFarm.enabled = not AutoFarm.enabled
        if AutoFarm.enabled then
            startAutoFarm()
        else
            stopAutoFarm()
        end
        setFarmStatus(AutoFarm.enabled)
    end
end)

warn("Standalone cargado — Tecla 5 para activar/desactivar")
