--[[=================================================================
                    ██╗     ██╗██████╗ ███████╗██╗   ██╗
                    ██║     ██║██╔══██╗╚══███╔╝╚██╗ ██╔╝
                    ██║     ██║██████╔╝  ███╔╝  ╚████╔╝
                    ██║     ██║██╔═══╝  ███╔╝    ╚██╔╝
                    ███████╗██║██║     ███████╗   ██║
                    ╚══════╝╚═╝╚═╝     ╚══════╝   ╚═╝
                     L I P Z Y   H U B   ·   Steal an Egg
====================================================================
  Script   : LIPZY HUB - Steal an Egg (Delta Executor / Luau)
  Version  : 1.0
  Features : [1] Anti Guard + Instant Steal + Auto Deliver (anti bug)
             [2] No Guard Aggro (freeze semua penjaga di 15 zona)
             [3] FPS Unlocker 120 + Smooth Graphics Optimization
  Cara Pakai: copy seluruh script -> paste di Delta Executor -> Execute
  Catatan   : script ini 100% client-side, semua perubahan otomatis
              dikembalikan (restore) saat fitur di-OFF / GUI di-close.
====================================================================
  PANDUAN SINGKAT
  ---------------
  FITUR 1 :
    1. ON kan "INSTANT STEAL + AUTO DELIVER"  -> auto cari telur, ambil,
       lalu terbang ke zona aman dan deliver otomatis (loop terus).
    2. ON kan "INSTANT PICK" kalau mau ambil telur manual: cukup 1x tap
       (HoldDuration dipaksa 0) tanpa jalan otomatis.
    3. "GODMODE / ANTI-RESET / ANTI-HIT" -> HP 9e9 + tombol reset dimatikan
       (kebal saat membawa telur).
    4. "FLY MODE (ANTI-BUG CARRY)" -> WAJIB ON (default) supaya telur
       dipindahkan memakai KECEPATAN (BodyVelocity) bukan blink/teleport,
       sehingga tidak pernah muncul "Delivery failed! The egg was returned
       to its nest." Kalau di-OFF, script otomatis pakai mode mikro-step
       (8-40 langkah kecil) yang juga aman.
    5. Kalau game tidak mendeteksi zona aman: berdiri di zona aman, lalu
       klik "Set Zona Aman di Sini (Manual)" dan ON kan fitur 1 lagi.

  FITUR 2 :
    ON kan "NO GUARD AGGRO" -> semua penjaga di 15 zona (Forest, Lake,
    Desert, Jungle, Snow, Volcano, Abyss Ocean, Prehistoric, Cosmic,
    Cherry Blossom, Titan Temple, Angel, Demons, Angel & Demons, dll)
    dibekukan di posisi aslinya (WalkSpeed 0 + kunci CFrame), dan
    otomatis di-restore saat toggle di-OFF.

  FITUR 3 :
    ON kan "FPS UNLOCKER 120" -> setfpscap(120) + kualitas grafik + culling
    partikel/trail/beam. Slider FPS Target bisa 30 - 240. "SMOOTH EKSTRA"
    mematikan lampu/bayangan/atmosphere untuk FPS maksimal.

  MOBILE / DELTA TIPS
  -------------------
  * Jendela = 80 x 90 unit GUI, dihitung otomatis dari ViewportSize
    (mobile ~ 330x370 px, desktop ~ 500x560 px) dan ikut berubah saat
    layar di-rotate / ukuran viewport berubah.
  * Semua toggle bisa di-tap di mana saja (row/track/thumb) dan bisa
    di-drag untuk memindahkan jendela. Tombol "-" = minimize, "X" = unload.
  * Tombol "STOP SEMUA FITUR" mematikan & me-restore semuanya tanpa
    menutup GUI.
====================================================================]]

--===================================================================
-- 0. ANTI DOUBLE-RUN (hapus instance lama kalau script dijalankan 2x)
--===================================================================
local GENV = (type(getgenv) == "function" and getgenv()) or _G
if type(GENV) == "table" and type(GENV.LIPZY_HUB) == "table" then
    local old = GENV.LIPZY_HUB
    if type(old.Unload) == "function" then
        pcall(old.Unload)
    end
end

--===================================================================
-- 1. CONFIG
--===================================================================
local CONFIG = {
    TITLE        = "LIPZY HUB",
    UNIT_W       = 80,      -- << lebar jendela GUI  = 80 unit
    UNIT_H       = 90,      -- << tinggi jendela GUI = 90 unit
    FPS_DEFAULT  = 120,     -- target FPS
    WALK_DEFAULT = 1000,    -- super speed
    FLY_DEFAULT  = 900,     -- kecepatan terbang
    PICK_RANGE   = 600,     -- jangkauan deteksi telur (stud), bisa diatur via slider
    MICRO_DELAY  = 0.18,    -- delay mikro anti "Delivery failed!"
    DELIVER_WAIT = 2.0,     -- tunggu konfirmasi delivery
    MAX_RETRY    = 3,       -- retry delivery sebelum menyerah
    GOD_HP       = 9e9,     -- HP godmode (hindari math.huge / inf)
}

--===================================================================
-- 2. SERVICES & STATE
--===================================================================
local Players      = game:GetService("Players")
local RunService   = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UIS          = game:GetService("UserInputService")
local StarterGui   = game:GetService("StarterGui")
local Lighting     = game:GetService("Lighting")
local Workspace    = game:GetService("Workspace")
local plr          = Players.LocalPlayer

local State = {
    instantPick   = false,   -- Fitur 1a: HoldDuration = 0 (1x tap)
    autoSteal     = false,   -- Fitur 1b: auto ambil + auto deliver
    godmode       = false,   -- Fitur 1c: kebal / anti reset
    superSpeed    = false,   -- Fitur 1d: WalkSpeed 1000
    flyCarry      = true,    -- Fitur 1e: terbang (velocity, anti bug)
    antiGuard     = false,   -- Fitur 2 : no guard aggro
    hardFreeze    = true,    -- Fitur 2 : kunci posisi penjaga
    fpsUnlock     = false,   -- Fitur 3 : FPS 120
    smoothExtra   = false,   -- Fitur 3 : matikan lampu/bayangan
    filterPrompt  = true,    -- hanya prompt telur yang di-instant
    walkSpeed     = CONFIG.WALK_DEFAULT,
    flySpeed      = CONFIG.FLY_DEFAULT,
    pickRange     = CONFIG.PICK_RANGE,
    fpsTarget     = CONFIG.FPS_DEFAULT,
}

--===================================================================
-- 3. UTILITIES
--===================================================================
local Connections = {}
local Cleanups    = {}

local function bind(signal, fn)
    local c = signal:Connect(fn)
    Connections[#Connections + 1] = c
    return c
end

local function create(className, props, parent)
    local inst = Instance.new(className)
    if props then
        for k, v in pairs(props) do
            pcall(function() inst[k] = v end)
        end
    end
    if parent then
        inst.Parent = parent
    end
    return inst
end

local function hasWord(str, words)
    if type(str) ~= "string" then return false end
    local low = string.lower(str)
    for i = 1, #words do
        if string.find(low, words[i], 1, true) then
            return true
        end
    end
    return false
end

local function notify(title, text, dur)
    pcall(function()
        StarterGui:SetCore("SendNotification", {
            Title = title,
            Text = text,
            Duration = dur or 4,
        })
    end)
end

local function getHum()
    local char = plr.Character
    if not char then return nil end
    local hum = char:FindFirstChildOfClass("Humanoid")
    if hum and hum.Health > 0 then return hum end
    return hum
end

local function getRoot()
    local char = plr.Character
    if not char then return nil end
    return char:FindFirstChild("HumanoidRootPart") or char.PrimaryPart
end

local function charAlive()
    local hum = getHum()
    return hum ~= nil and hum.Health > 0
end

-- status bar (di-set saat UI dibangun)
local StatusLabel, StatusFps
local function setStatus(text)
    if StatusLabel then
        pcall(function() StatusLabel.Text = text end)
    end
end

-- forward declaration: fungsi unload global (dipakai tombol X)
local unload

--===================================================================
-- 4. DEFINISI ZONA & PENJAGA (Fitur 2)
--===================================================================
local ZONE_NAMES = {
    "forest", "hutan", "lake", "danau", "desert", "gurun", "jungle",
    "snow", "ice", "salju", "volcano", "gunung api", "abyss", "ocean",
    "laut", "prehistoric", "prasejarah", "cosmic", "cosmos", "cherry",
    "blossom", "sakura", "titan", "temple", "candi", "angel", "malaikat",
    "demon", "demon", "iblis", "island", "pulau", "cave", "gua",
    "mountain", "space", "nebula", "galaxy",
}

local GUARD_WORDS = {
    "guard", "penjaga", "goon", "thief", "bandit", "hunter", "sentinel",
    "watcher", "monster", "enemy", "killer", "brute", "zombie", "wolf",
    "bear", "dragon", "boss", "npc", "police", "officer", "security",
    "protector", "keeper", "soldier", "trap", "turret", "minion", "chaser",
    "pursuer", "raider", "pirate", "knight", "robot",
}

local EGG_WORDS = {
    "egg", "eggs", "telur", "steal", "stolen", "nest", "sarang", "carry",
    "bawa", "amber", "dino egg", "pet egg",
}

local PICK_WORDS = {
    "steal", "take", "grab", "pick", "ambil", "carry", "bawa", "collect",
    "open", "hatch", "hold", "pegang",
}

--===================================================================
-- 5. FLY ENGINE (velocity based => Touched tetap valid, anti bug delivery)
--===================================================================
local Fly = {
    active = false,
    bv = nil,
    bg = nil,
    gravity = nil,
    platformStand = false,
    savedWalkSpeed = nil,
}

function Fly:Start()
    local root, hum = getRoot(), getHum()
    if not root then return false end
    if self.active then return true end
    local oldBV = root:FindFirstChild("LIPZY_FLY_BV")
    if oldBV then pcall(function() oldBV:Destroy() end) end
    local oldBG = root:FindFirstChild("LIPZY_FLY_BG")
    if oldBG then pcall(function() oldBG:Destroy() end) end

    self.bv = create("BodyVelocity", {
        Name = "LIPZY_FLY_BV",
        P = 12500,
        MaxForce = Vector3.new(1e5, 1e5, 1e5),
        Velocity = Vector3.new(0, 0, 0),
    }, root)

    self.bg = create("BodyGyro", {
        Name = "LIPZY_FLY_BG",
        P = 10000,
        D = 500,
        MaxTorque = Vector3.new(1e5, 1e5, 1e5),
        CFrame = root.CFrame,
    }, root)

    if hum then
        self.platformStand = hum.PlatformStand
        pcall(function() hum.PlatformStand = true end)
        pcall(function() hum.AutoRotate = false end)
    end
    self.gravity = Workspace.Gravity
    pcall(function() Workspace.Gravity = 0 end)
    self.active = true
    return true
end

function Fly:Stop()
    if not self.active then return end
    self.active = false
    if self.bv then
        pcall(function() self.bv:Destroy() end)
        self.bv = nil
    end
    if self.bg then
        pcall(function() self.bg:Destroy() end)
        self.bg = nil
    end
    local hum = getHum()
    if hum then
        pcall(function() hum.PlatformStand = self.platformStand end)
        pcall(function() hum.AutoRotate = true end)
    end
    if self.gravity then
        pcall(function() Workspace.Gravity = self.gravity end)
    end
end

-- terbang ke sebuah Vector3 dengan kecepatan bertahap (aman untuk anti-cheat game)
local function flyTo(goal, speed, stopDist, maxTime)
    local root = getRoot()
    if not root or not Fly.active then return false end
    speed = speed or State.flySpeed
    stopDist = stopDist or 3.5
    maxTime = maxTime or 10
    local t0 = os.clock()
    local reached = false
    while Fly.active do
        local myRoot = getRoot()
        if not myRoot or myRoot ~= root then break end
        local pos = root.Position
        local delta = goal - pos
        local dist = delta.Magnitude
        if dist <= stopDist then
            reached = true
            break
        end
        local vel = delta.Unit * math.min(speed, dist * 14)
        pcall(function() Fly.bv.Velocity = vel end)
        pcall(function() Fly.bg.CFrame = CFrame.new(pos, goal) end)
        if os.clock() - t0 > maxTime then break end
        RunService.Heartbeat:Wait()
    end
    if Fly.bv then
        pcall(function() Fly.bv.Velocity = Vector3.zero end)
    end
    return reached
end

-- mikro-step teleport (fallback kalau Fly Mode OFF, tetap "halus" & replikasi valid)
local function microTravel(goal, maxSteps)
    local root = getRoot()
    if not root then return false end
    local start = root.Position
    local total = (goal - start).Magnitude
    if total < 1 then return true end
    local steps = math.clamp(math.floor(total / 14), 1, maxSteps or 40)
    for i = 1, steps do
        local r = getRoot()
        if not r then return false end
        local p = start:Lerp(goal, i / steps)
        pcall(function() r.CFrame = CFrame.new(p) end)
        task.wait(0.035)
    end
    return true
end

-- perpindahan universal: pakai Fly kalau aktif, kalau tidak pakai mikro-step
local function travelTo(goal, speed, stopDist)
    if State.flyCarry then
        if Fly:Start() then
            return flyTo(goal, speed, stopDist)
        end
    end
    return microTravel(goal)
end

--===================================================================
-- 6. FITUR 1A - INSTANT EGG PICK (ProximityPrompt HoldDuration = 0)
--===================================================================
local PromptSnapshot = {}   -- [prompt] = table props asli
local PromptPool     = {}   -- daftar prompt telur
local PromptCache    = {}   -- [prompt] = { pos = Vector3, t = os.clock() }
local PromptFail     = {}   -- [prompt] = os.clock() (prompt yang gagal -> di-skip sementara)

local PROMPT_PROPS = {
    "HoldDuration", "Enabled", "RequiresLineOfSight",
    "MaxActivationDistance", "ClickablePrompt",
}

local function applyPromptNow(p)
    pcall(function() p.HoldDuration = 0 end)
    pcall(function() p.RequiresLineOfSight = false end)
    pcall(function() p.Enabled = true end)
    pcall(function()
        if p.MaxActivationDistance < 60 then
            p.MaxActivationDistance = 60
        end
    end)
end

local function isEggPrompt(p)
    if not p.Parent then return false end
    if p.ObjectText ~= "" and (hasWord(p.ObjectText, EGG_WORDS) or hasWord(p.ObjectText, PICK_WORDS)) then
        return true
    end
    if p.ActionText ~= "" and (hasWord(p.ActionText, EGG_WORDS) or hasWord(p.ActionText, PICK_WORDS)) then
        return true
    end
    local node, depth = p.Parent, 0
    while node and node ~= Workspace and depth < 3 do
        if hasWord(node.Name, EGG_WORDS) then return true end
        node = node.Parent
        depth = depth + 1
    end
    return false
end

local function applyPrompt(p)
    if not p.Parent then return end
    if not PromptSnapshot[p] then
        local snap = {}
        for i = 1, #PROMPT_PROPS do
            local k = PROMPT_PROPS[i]
            local ok, v = pcall(function() return p[k] end)
            if ok then snap[k] = v end
        end
        PromptSnapshot[p] = snap
        PromptPool[#PromptPool + 1] = p
    end
    -- INSTANT: tidak perlu menahan lama (1x tap langsung trigger)
    applyPromptNow(p)
end

local function restorePrompt(p)
    local snap = PromptSnapshot[p]
    if snap and p.Parent then
        for k, v in pairs(snap) do
            pcall(function() p[k] = v end)
        end
    end
    PromptSnapshot[p] = nil
end

local lastPromptScan = 0

-- scan penuh (berat) -> dibatasi minimal 6 detik sekali
local function scanPrompts(force)
    if not force and (os.clock() - lastPromptScan) < 6 then
        return #PromptPool
    end
    lastPromptScan = os.clock()
    local found = 0
    local list = Workspace:GetDescendants()
    for i = 1, #list do
        local d = list[i]
        if d:IsA("ProximityPrompt") then
            if (not State.filterPrompt) or isEggPrompt(d) then
                applyPrompt(d)
                found = found + 1
            end
        end
    end
    return found
end

-- re-apply ringan (hanya prompt yang sudah terdaftar) -> aman dipanggil sering
local function reapplyPrompts()
    local alive = 0
    for i = #PromptPool, 1, -1 do
        local p = PromptPool[i]
        if not p.Parent then
            PromptSnapshot[p] = nil
            table.remove(PromptPool, i)
        else
            applyPromptNow(p)
            alive = alive + 1
        end
    end
    return alive
end

local function cleanupPrompts()
    for p in pairs(PromptSnapshot) do
        restorePrompt(p)
    end
    PromptSnapshot = {}
    PromptPool = {}
    PromptCache = {}
    PromptFail = {}
    lastPromptScan = 0
end

local function getPromptPos(p)
    local obj = p.Parent
    if not obj then return nil end
    if obj:IsA("BasePart") then return obj.Position end
    if obj:IsA("Attachment") then return obj.WorldPosition end
    if obj:IsA("Model") and obj.PrimaryPart then return obj.PrimaryPart.Position end
    local part = obj:FindFirstChildWhichIsA("BasePart", true)
    if part then return part.Position end
    return nil
end

-- trigger prompt tanpa hold (1x tap) + fallback fireproximityprompt (Delta)
local function triggerPrompt(p)
    if not p or not p.Parent then return false end
    pcall(function() p.HoldDuration = 0 end)
    local ok1 = false
    if type(fireproximityprompt) == "function" then
        ok1 = pcall(fireproximityprompt, p)
    end
    local ok2 = pcall(function()
        p:InputHoldBegin()
        task.wait()
        p:InputHoldEnd()
    end)
    return ok1 or ok2
end

-- posisi prompt di-cache supaya loop pencarian tetap ringan (no lag)
local function cachedPromptPos(p)
    local now = os.clock()
    local c = PromptCache[p]
    if c and (now - c.t) < 0.8 then
        return c.pos
    end
    local pos = getPromptPos(p)
    if pos then
        PromptCache[p] = { pos = pos, t = now }
    else
        PromptCache[p] = nil
    end
    return pos
end

local function nearestEggPrompt(range)
    local root = getRoot()
    if not root then return nil, math.huge end
    local myPos = root.Position
    local now = os.clock()
    local best, bestDist = nil, range or CONFIG.PICK_RANGE
    for i = 1, #PromptPool do
        local p = PromptPool[i]
        if p.Parent then
            local failedAt = PromptFail[p]
            if not failedAt or (now - failedAt) > 8 then
                local pos = cachedPromptPos(p)
                if pos then
                    local d = (pos - myPos).Magnitude
                    if d < bestDist then
                        best, bestDist = p, d
                    end
                end
            end
        end
    end
    return best, bestDist
end

--===================================================================
-- 7. DETEKSI TELUR YANG DIBAWA
--===================================================================
local function snapshotTools(char)
    local t = {}
    if not char then return t end
    for _, c in ipairs(char:GetChildren()) do
        if c:IsA("Tool") then t[c] = true end
    end
    return t
end

local CARRY_ATTRS = { "HasEgg", "Carrying", "Carry", "IsCarrying", "HoldEgg", "HasCarry" }

local function detectCarry(char, before)
    if not char then return nil end
    -- 1) Tool BARU di character (hanya kalau ada snapshot "sebelum ambil")
    if before then
        for _, c in ipairs(char:GetChildren()) do
            if c:IsA("Tool") and not before[c] then
                return c
            end
        end
    end
    -- 1b) Tool yang namanya mengandung "egg"/"telur"
    for _, c in ipairs(char:GetChildren()) do
        if c:IsA("Tool") and hasWord(c.Name, EGG_WORDS) then
            return c
        end
    end
    -- 2) Attribute (beberapa game pakai attribute bukan Tool)
    local hum = char:FindFirstChildOfClass("Humanoid")
    for i = 1, #CARRY_ATTRS do
        local a = CARRY_ATTRS[i]
        local v = char:GetAttribute(a)
        if v == nil and hum then v = hum:GetAttribute(a) end
        if v == true then return true end
    end
    -- 3) Part/Attachment bawa telur (nama mengandung "egg"/"telur")
    for _, d in ipairs(char:GetDescendants()) do
        if (d:IsA("BasePart") or d:IsA("Attachment") or d:IsA("Model")) and hasWord(d.Name, EGG_WORDS) then
            return d
        end
    end
    return nil
end

local function stillCarrying(carry)
    if carry == nil then return false end
    if carry == true then
        local char = plr.Character
        return detectCarry(char, nil) ~= nil
    end
    if typeof(carry) == "Instance" then
        return carry.Parent ~= nil
    end
    return false
end

local function waitForCarry(char, before, timeout)
    local t0 = os.clock()
    while os.clock() - t0 < (timeout or 1.5) do
        if not char or char ~= plr.Character then return nil end
        local c = detectCarry(char, before)
        if c then return c end
        task.wait(0.05)
    end
    return nil
end

--===================================================================
-- 8. DETEKSI ZONA AMAN (Safe Zone / Delivery Zone)
--===================================================================
local SafeZone = { part = nil, manual = nil, lastScan = 0 }

local SAFE_STRONG = {
    "deliver", "safe", "sale", "sell", "submit", "deposit", "drop",
    "collect", "bank", "vault", "zonadeliver", "zonaaman",
}
local SAFE_WORDS = {
    "delivery", "deliver", "safe", "safezone", "safe zone", "sale", "sell",
    "submit", "deposit", "dropoff", "drop zone", "collect", "collection",
    "bank", "vault", "storage", "zonaaman", "zona aman", "aman", "base",
    "home", "tent", "camp", "house", "trade", "shop", "store", "exit",
}

local QUICK_ZONE_NAMES = {
    "SafeZone", "safezone", "SAFEZONE", "Safe_Zone", "safe_zone",
    "DeliveryZone", "deliveryzone", "DeliverZone", "deliverzone",
    "SaleZone", "salezone", "SellZone", "sellzone", "SellZone",
    "SubmitZone", "DropZone", "dropzone", "EggDelivery", "ZonaAman",
    "BankZone", "Vault", "Delivery", "Safe", "Sale",
}

local function scoreZonePart(part, extraNames)
    local score = 0
    local names = { part.Name }
    if extraNames then
        for i = 1, #extraNames do names[#names + 1] = extraNames[i] end
    end
    for i = 1, #names do
        local low = string.lower(names[i])
        for j = 1, #SAFE_STRONG do
            if string.find(low, SAFE_STRONG[j], 1, true) then
                score = score + 14
                break
            end
        end
        for j = 1, #SAFE_WORDS do
            if string.find(low, SAFE_WORDS[j], 1, true) then
                score = score + 5
                break
            end
        end
    end
    local vol = part.Size.X * part.Size.Y * part.Size.Z
    if vol > 500 then score = score + 2 end
    if vol > 5000 then score = score + 2 end
    if not part.CanCollide then score = score + 1 end
    local root = getRoot()
    if root and (part.Position - root.Position).Magnitude < 600 then
        score = score + 1
    end
    return score
end

local function findSafeZone()
    if SafeZone.manual then return SafeZone.manual, 999 end
    local best, bestScore = nil, 0

    local function consider(part, names)
        if not part or not part.Parent then return end
        if not part:IsA("BasePart") then return end
        if plr.Character and part:IsDescendantOf(plr.Character) then return end
        local s = scoreZonePart(part, names)
        if s > bestScore then
            bestScore = s
            best = part
        end
    end

    -- (a) cari nama terkenal dulu (cepat, engine-side search)
    for i = 1, #QUICK_ZONE_NAMES do
        local obj = Workspace:FindFirstChild(QUICK_ZONE_NAMES[i], true)
        if obj then
            if obj:IsA("BasePart") then
                consider(obj, nil)
            elseif obj:IsA("Model") or obj:IsA("Folder") then
                if obj.PrimaryPart then consider(obj.PrimaryPart, { obj.Name }) end
                local kids = obj:GetDescendants()
                for k = 1, #kids do
                    if kids[k]:IsA("BasePart") then consider(kids[k], { obj.Name }) end
                end
            end
        end
    end

    -- (b) pindai anak level atas Workspace yang namanya mengandung kata kunci zona
    local tops = Workspace:GetChildren()
    for i = 1, #tops do
        local top = tops[i]
        if top:IsA("BasePart") then
            consider(top, nil)
        elseif (top:IsA("Model") or top:IsA("Folder")) and hasWord(top.Name, SAFE_WORDS) then
            local kids = top:GetDescendants()
            for k = 1, #kids do
                if kids[k]:IsA("BasePart") then consider(kids[k], { top.Name }) end
            end
        end
    end

    -- (c) fallback terakhir: pindai semua part (dibatasi waktu, max 1.5 detik)
    if bestScore < 10 then
        local t0 = os.clock()
        local all = Workspace:GetDescendants()
        for i = 1, #all do
            local d = all[i]
            if d:IsA("BasePart") then consider(d, nil) end
            if (i % 400) == 0 and (os.clock() - t0) > 1.5 then break end
        end
    end

    SafeZone.part = best
    SafeZone.lastScan = os.clock()
    return best, bestScore
end

--===================================================================
-- 9. FITUR 1B - SAFE AUTO DELIVERY (anti "Delivery failed!")
--===================================================================
local Stats = { delivered = 0, failed = 0, eggs = 0 }

local function getEggCount()
    local ls = plr:FindFirstChild("leaderstats")
    if ls then
        for _, v in ipairs(ls:GetChildren()) do
            if (v:IsA("IntValue") or v:IsA("NumberValue"))
                and hasWord(v.Name, { "egg", "telur", "steal", "stolen", "collect", "eggcount" }) then
                return v.Value, v.Name
            end
        end
    end
    for _, k in ipairs({ "Eggs", "EggCount", "StolenEggs", "Egg" }) do
        local v = plr:GetAttribute(k)
        if type(v) == "number" then return v, "attr:" .. k end
    end
    return nil, nil
end

local function insideZone(part, pos)
    local localPos = part.CFrame:PointToObjectSpace(pos)
    return math.abs(localPos.X) <= part.Size.X * 0.5 + 2
        and math.abs(localPos.Y) <= part.Size.Y * 0.5 + 3
        and math.abs(localPos.Z) <= part.Size.Z * 0.5 + 2
end

-- simulasi "jalan" di dalam zona supaya Touched / region check server benar-benar aktif
local function walkSimulate(zonePos, duration)
    local hum = getHum()
    if not hum then return end
    local savedWS = hum.WalkSpeed
    pcall(function() hum.WalkSpeed = 16 end)
    local t0 = os.clock()
    while os.clock() - t0 < duration do
        local root = getRoot()
        if not root or not charAlive() then break end
        local delta = zonePos - root.Position
        delta = Vector3.new(delta.X, 0, delta.Z)
        if delta.Magnitude > 0.6 then
            pcall(function() hum:Move(delta.Unit) end)
        else
            -- jitter kecil: memicu Touched berulang tanpa keluar zona
            pcall(function() hum:Move(Vector3.new(math.random() - 0.5, 0, math.random() - 0.5).Unit) end)
        end
        RunService.Heartbeat:Wait()
    end
    pcall(function()
        hum:Move(Vector3.zero)
        hum.WalkSpeed = savedWS
    end)
end

-- satu percobaan delivery yang "bersih" (bukan blink/teleport langsung)
local function attemptDelivery(carry)
    local zone = SafeZone.part
    if not zone or not zone.Parent then
        zone = findSafeZone()
    end
    if not zone then return false, "nozone" end

    local zPos, zSize = zone.Position, zone.Size
    local root = getRoot()
    if not root then return false, "nochar" end

    -- (1) mendekat dari LUAR zona dulu (server butuh melihat player masuk, bukan muncul)
    local outside = Vector3.new(
        zPos.X + math.max(zSize.X * 0.5, 10) + 14,
        zPos.Y + math.max(zSize.Y * 0.5, 4) + 10,
        zPos.Z + math.max(zSize.Z * 0.5, 10) + 14
    )
    travelTo(outside, State.flySpeed, 6)
    task.wait(CONFIG.MICRO_DELAY * 0.5)

    -- (2) hover tepat di atas zona
    local hover = Vector3.new(zPos.X, zPos.Y + math.max(zSize.Y * 0.5, 4) + 7, zPos.Z)
    travelTo(hover, State.flySpeed * 0.7, 4)
    task.wait(CONFIG.MICRO_DELAY)

    -- (3) turun perlahan pakai KECEPATAN (bukan set CFrame) => Touched valid
    if Fly.active then
        local t0 = os.clock()
        while Fly.active and (os.clock() - t0) < 4 do
            local r = getRoot()
            if not r then break end
            local d = zPos - r.Position
            if d.Magnitude <= 2.5 then break end
            local vel = d.Unit * math.min(95, d.Magnitude * 6)
            pcall(function() Fly.bv.Velocity = vel end)
            RunService.Heartbeat:Wait()
        end
        Fly:Stop()      -- gravitasi normal -> mendarat alami di zona
    else
        microTravel(zPos + Vector3.new(0, 3, 0))
    end
    task.wait(0.15)

    -- (4) mikro-step kecil masuk tepat ke volume zona (step 1/8, sangat halus)
    local r = getRoot()
    if r and not insideZone(zone, r.Position) then
        local startPos = r.Position
        for i = 1, 8 do
            local rr = getRoot()
            if not rr then break end
            local p = startPos:Lerp(zPos, i / 8)
            pcall(function() rr.CFrame = CFrame.new(p) end)
            task.wait(0.04)
        end
    end

    -- (5) simulasi jalan di dalam zona (Touched & region check terpenuhi)
    task.wait(CONFIG.MICRO_DELAY)
    walkSimulate(zPos, 0.65)

    -- (6) tunggu konfirmasi tanpa bergerak (jangan lari keluar sebelum sukses!)
    local t1 = os.clock()
    while (os.clock() - t1) < CONFIG.DELIVER_WAIT do
        if not stillCarrying(carry) then return true end
        task.wait(0.08)
    end

    -- (7) usaha terakhir: maju-mundur halus di dalam zona
    local hum = getHum()
    if hum then
        pcall(function() hum.WalkSpeed = 16 end)
        for i = 1, 6 do
            pcall(function() hum:Move(Vector3.new(math.random() - 0.5, 0, math.random() - 0.5).Unit) end)
            task.wait(0.08)
        end
        pcall(function()
            hum:Move(Vector3.zero)
            hum.WalkSpeed = State.superSpeed and State.walkSpeed or 16
        end)
    end

    return not stillCarrying(carry)
end

local function deliverRoutine(carry)
    local countBefore = getEggCount()
    for attempt = 1, CONFIG.MAX_RETRY do
        setStatus("Delivering... percobaan " .. attempt .. "/" .. CONFIG.MAX_RETRY)
        local ok = false
        pcall(function() ok = attemptDelivery(carry) end)
        if ok then
            Stats.delivered = Stats.delivered + 1
            local countAfter = getEggCount()
            if countBefore and countAfter then
                Stats.eggs = countAfter
            end
            setStatus("Delivery sukses (" .. Stats.delivered .. "x) | Failed: " .. Stats.failed)
            return true
        end
        Stats.failed = Stats.failed + 1
        setStatus("Delivery gagal, retry... | Failed: " .. Stats.failed)
        task.wait(0.3)
    end
    return false
end

--===================================================================
-- 10. FITUR 1 - MASTER LOOP (Instant Steal + Auto Deliver)
--===================================================================
local StealJob = { running = false, thread = nil, lastScan = 0 }

local function doStealCycle()
    local char = plr.Character
    local root = getRoot()
    if not char or not root or not charAlive() then
        task.wait(0.4)
        return
    end

    -- refresh prompt berkala (telur respawn -> prompt baru)
    reapplyPrompts()
    if os.clock() - StealJob.lastScan > 8 then
        StealJob.lastScan = os.clock()
        pcall(function() scanPrompts(false) end)
    end

    -- sudah bawa telur? langsung deliver
    local carried = detectCarry(char, nil)
    if carried then
        setStatus("Membawa telur -> menuju zona aman...")
        deliverRoutine(carried)
        task.wait(0.2)
        return
    end

    local prompt = nearestEggPrompt(State.pickRange or CONFIG.PICK_RANGE)
    if not prompt then
        setStatus("Mencari telur...")
        task.wait(0.35)
        return
    end

    -- (1) pindah ke telur (velocity / mikro-step, bukan blink liar)
    local pos = getPromptPos(prompt)
    if not pos then
        task.wait(0.2)
        return
    end
    setStatus("Mengambil telur...")
    travelTo(pos + Vector3.new(0, 2.6, 0), State.flySpeed, 3.2)
    task.wait(CONFIG.MICRO_DELAY)   -- delay mikro: beri waktu replikasi posisi ke server

    -- (2) trigger prompt cukup 1x tap (HoldDuration sudah 0)
    local before = snapshotTools(char)
    local carry = nil
    for i = 1, 3 do
        triggerPrompt(prompt)
        carry = waitForCarry(char, before, 0.6)
        if carry then break end
        task.wait(0.1)
    end

    if not carry then
        setStatus("Gagal ambil telur, cari telur lain...")
        -- prompt ini kemungkinan bukan telur / sudah diambil -> skip sementara 8 detik
        PromptFail[prompt] = os.clock()
        task.wait(0.25)
        return
    end

    PromptFail[prompt] = nil

    -- (3) delay mikro sebelum delivery (kunci anti "Delivery failed!")
    task.wait(CONFIG.MICRO_DELAY)

    -- (4) Auto teleport / fly carry ke zona aman
    if State.flyCarry or State.superSpeed or State.autoSteal then
        setStatus("Telur didapat -> terbang ke zona aman...")
        deliverRoutine(carry)
    end
    task.wait(0.1)
end

local function startStealLoop()
    if StealJob.running then return end
    StealJob.running = true
    StealJob.lastScan = 0
    pcall(function() scanPrompts(true) end)
    if not SafeZone.part then pcall(findSafeZone) end
    if not SafeZone.part then
        notify("LIPZY HUB", "Zona aman belum terdeteksi. Klik 'Rescan Zona Aman' atau 'Set Zona Aman di Sini'.", 6)
    end
    StealJob.thread = task.spawn(function()
        while StealJob.running do
            local ok, err = pcall(doStealCycle)
            if not ok then
                setStatus("Retry... (" .. tostring(err):sub(1, 40) .. ")")
                task.wait(0.5)
            end
        end
        Fly:Stop()
    end)
end

local function stopStealLoop()
    StealJob.running = false
    Fly:Stop()
    setStatus("Idle")
end

--===================================================================
-- 11. FITUR 1C - GODMODE / ANTI RESET / ANTI HIT
--===================================================================
local God = { loopRunning = false }

local function applyGodmode()
    local hum = getHum()
    if not hum then return end
    pcall(function()
        if hum.MaxHealth < CONFIG.GOD_HP then
            hum.MaxHealth = CONFIG.GOD_HP
        end
        if hum.Health < CONFIG.GOD_HP then
            hum.Health = CONFIG.GOD_HP
        end
        hum.BreakJointsOnDeath = false
        hum:SetStateEnabled(Enum.HumanoidStateType.FallingDown, false)
        hum:SetStateEnabled(Enum.HumanoidStateType.Ragdoll, false)
    end)
end

local function startGodmode()
    if God.loopRunning then return end
    God.loopRunning = true
    applyGodmode()
    pcall(function() StarterGui:SetCore("ResetButtonCallback", false) end)
    task.spawn(function()
        while God.loopRunning do
            if State.godmode then
                applyGodmode()
            end
            task.wait(0.15)
        end
    end)
end

local function stopGodmode()
    God.loopRunning = false
    pcall(function() StarterGui:SetCore("ResetButtonCallback", true) end)
    local hum = getHum()
    if hum then
        pcall(function()
            hum.MaxHealth = 100
            if hum.Health > 100 then hum.Health = 100 end
            hum:SetStateEnabled(Enum.HumanoidStateType.FallingDown, true)
            hum:SetStateEnabled(Enum.HumanoidStateType.Ragdoll, true)
        end)
    end
end

--===================================================================
-- 12. FITUR 1D - SUPER SPEED 1000
--===================================================================
local Speed = { savedKey = nil, savedValue = nil }

local function applyWalkSpeed()
    local hum = getHum()
    if not hum then return end
    if Speed.savedKey ~= plr.Character then
        Speed.savedKey = plr.Character
        Speed.savedValue = hum.WalkSpeed
    end
    if State.superSpeed then
        pcall(function()
            hum.WalkSpeed = State.walkSpeed
            hum.JumpPower = 120
            hum.JumpHeight = 60
            hum.UseJumpPower = true
        end)
    else
        local back = Speed.savedValue or 16
        if back < 8 then back = 16 end
        pcall(function()
            hum.WalkSpeed = back
            hum.JumpPower = 50
        end)
    end
end

--===================================================================
-- 13. FITUR 2 - NO GUARD AGGRO (FREEZE PENJAGA)
--===================================================================
local Guards = {
    frozen = {},
    running = false,
    scanned = 0,
}

local function isGuardHumanoid(hum)
    local model = hum.Parent
    if not model or not model:IsA("Model") then return false end
    if plr.Character and model == plr.Character then return false end
    if Players:GetPlayerFromCharacter(model) then return false end
    if hasWord(model.Name, GUARD_WORDS) then return true end
    -- NPC dengan senjata (weapon/tool) di dalam model = sangat mungkin penjaga
    local tool = model:FindFirstChildWhichIsA("Tool")
    if tool and hasWord(tool.Name, { "sword", "gun", "weapon", "blade", "spear", "axe", "bat", "stick", "net", "taser", "bow" }) then
        return true
    end
    local node, depth = model.Parent, 0
    while node and node ~= Workspace and depth < 4 do
        if hasWord(node.Name, GUARD_WORDS) then return true end
        if hasWord(node.Name, ZONE_NAMES) then return true end
        node = node.Parent
        depth = depth + 1
    end
    return false
end

local function freezeGuard(hum)
    if Guards.frozen[hum] then return end
    local model = hum.Parent
    if not model then return end
    local root = model:FindFirstChild("HumanoidRootPart")
    local entry = {
        ws = hum.WalkSpeed,
        jp = hum.JumpPower,
        jh = hum.JumpHeight,
        autoRotate = hum.AutoRotate,
        root = root,
        anchor = root and root.Position or nil,
    }
    Guards.frozen[hum] = entry
    pcall(function()
        hum.WalkSpeed = 0
        hum.JumpPower = 0
        hum.JumpHeight = 0
        hum.AutoRotate = false
        if entry.anchor then hum:MoveTo(entry.anchor) end
    end)
end

local function unfreezeGuard(hum, entry)
    pcall(function()
        hum.WalkSpeed = entry.ws or 12
        hum.JumpPower = entry.jp or 50
        hum.JumpHeight = entry.jh or 7
        hum.AutoRotate = entry.autoRotate ~= false
    end)
    Guards.frozen[hum] = nil
end

local function scanGuards()
    local n = 0
    local list = Workspace:GetDescendants()
    for i = 1, #list do
        local d = list[i]
        if d:IsA("Humanoid") and d.Health > 0 and not Guards.frozen[d] then
            if isGuardHumanoid(d) then
                freezeGuard(d)
                n = n + 1
            end
        end
    end
    Guards.scanned = n
    return n
end

local function startGuardLoop()
    if Guards.running then return end
    Guards.running = true
    setStatus("Scan penjaga...")
    task.spawn(function()
        local lastScan = 0
        while Guards.running do
            if State.antiGuard then
                local now = os.clock()
                -- scan penuh hanya tiap 1.5 detik (penjaga baru sudah ditangkap DescendantAdded)
                if (now - lastScan) > 1.5 then
                    lastScan = now
                    pcall(scanGuards)
                end
                -- re-apply tiap tick (server bisa menimpa WalkSpeed / posisi)
                for hum, entry in pairs(Guards.frozen) do
                    if not hum.Parent or hum.Health <= 0 then
                        Guards.frozen[hum] = nil
                    else
                        pcall(function()
                            if hum.WalkSpeed ~= 0 then hum.WalkSpeed = 0 end
                            if hum.JumpPower ~= 0 then hum.JumpPower = 0 end
                            if hum.JumpHeight ~= 0 then hum.JumpHeight = 0 end
                            hum:Move(Vector3.zero)
                        end)
                        if State.hardFreeze and entry.root and entry.root.Parent and entry.anchor then
                            if (entry.root.Position - entry.anchor).Magnitude > 1 then
                                pcall(function() entry.root.CFrame = CFrame.new(entry.anchor) end)
                            end
                        end
                    end
                end
                task.wait(0.1)
            else
                task.wait(0.35)
            end
        end
    end)
end

local function stopGuardLoop()
    Guards.running = false
    for hum, entry in pairs(Guards.frozen) do
        if hum.Parent and hum.Health > 0 then
            unfreezeGuard(hum, entry)
        else
            Guards.frozen[hum] = nil
        end
    end
    Guards.frozen = {}
end

--===================================================================
-- 14. FITUR 3 - FPS UNLOCKER 120 + SMOOTH GRAPHICS
--===================================================================
local Perf = {
    effects = {},
    lights = {},
    lighting = {},
    applied = false,
    extraApplied = false,
    culled = 0,
}

local EFFECT_CLASSES = { "ParticleEmitter", "Trail", "Beam", "Fire", "Smoke", "Sparkles" }

local function isEffect(inst)
    for i = 1, #EFFECT_CLASSES do
        if inst:IsA(EFFECT_CLASSES[i]) then return true end
    end
    return false
end

local function cullEffects()
    local n = 0
    local list = Workspace:GetDescendants()
    for i = 1, #list do
        local d = list[i]
        if isEffect(d) and not Perf.effects[d] then
            local ok, v = pcall(function() return d.Enabled end)
            if ok then
                Perf.effects[d] = v
                pcall(function() d.Enabled = false end)
                n = n + 1
            end
        end
    end
    Perf.culled = Perf.culled + n
    return n
end

local function restoreEffects()
    for inst, v in pairs(Perf.effects) do
        if inst.Parent then
            pcall(function() inst.Enabled = v end)
        end
    end
    Perf.effects = {}
    Perf.culled = 0
end

local function setFpsCap(n)
    local applied = false
    if type(setfpscap) == "function" then
        applied = pcall(setfpscap, n) or applied
    end
    if not applied and type(setfflag) == "function" then
        pcall(setfflag, "TaskSchedulerTargetFps", tostring(n))
        applied = true
    end
    -- fallback: paksa kualitas grafis ke level terendah (membuka headroom FPS)
    pcall(function()
        local UGS = UserSettings():GetService("UserGameSettings")
        UGS.SavedQualityLevel = Enum.SavedQualitySetting.QualityLevel1
    end)
    return applied
end

local function applySmoothExtra(on)
    if on then
        if not Perf.extraApplied then
            Perf.extraApplied = true
            Perf.lighting.GlobalShadows = Lighting.GlobalShadows
            Perf.lighting.FogEnd = Lighting.FogEnd
            Perf.lighting.Brightness = Lighting.Brightness
            pcall(function()
                Lighting.GlobalShadows = false
                Lighting.FogEnd = 1e6
                Lighting.Brightness = 2
                Workspace.Terrain.WaterWaveSize = 0
                Workspace.Terrain.WaterWaveSpeed = 0
            end)
            for _, d in ipairs(Lighting:GetChildren()) do
                if d:IsA("Atmosphere") or d:IsA("BloomEffect") or d:IsA("SunRaysEffect")
                    or d:IsA("DepthOfFieldEffect") or d:IsA("ColorCorrectionEffect") then
                    Perf.lighting[d] = d.Enabled
                    pcall(function() d.Enabled = false end)
                end
            end
            local list = Workspace:GetDescendants()
            for i = 1, #list do
                local d = list[i]
                if d:IsA("Light") then
                    Perf.lights[d] = d.Enabled
                    pcall(function() d.Enabled = false end)
                end
            end
        end
    else
        if Perf.extraApplied then
            Perf.extraApplied = false
            pcall(function()
                Lighting.GlobalShadows = Perf.lighting.GlobalShadows == true
                Lighting.FogEnd = Perf.lighting.FogEnd or 100000
                Lighting.Brightness = Perf.lighting.Brightness or 2
            end)
            for inst, v in pairs(Perf.lighting) do
                if typeof(inst) == "Instance" and inst.Parent then
                    pcall(function() inst.Enabled = v end)
                end
            end
            for inst, v in pairs(Perf.lights) do
                if inst.Parent then
                    pcall(function() inst.Enabled = v end)
                end
            end
            Perf.lighting = {}
            Perf.lights = {}
        end
    end
end

local function enableFps()
    if Perf.applied then return end
    Perf.applied = true
    setFpsCap(State.fpsTarget)
    cullEffects()
    setStatus("FPS Unlocker ON -> " .. tostring(State.fpsTarget) .. " FPS")
end

local function disableFps()
    Perf.applied = false
    setFpsCap(60)
    restoreEffects()
    setStatus("FPS Unlocker OFF")
end

--===================================================================
-- 15. UI LIBRARY (LIPZY HUB) - responsif untuk Mobile / Delta
--===================================================================
local UI = {
    S = 4,
    W = 320,
    H = 360,
    hooks = {},
    toggles = {},
    sliders = {},
    minimized = false,
}

local THEME = {
    bg1      = Color3.fromRGB(16, 16, 24),
    bg2      = Color3.fromRGB(26, 26, 38),
    bg3      = Color3.fromRGB(38, 38, 54),
    accent   = Color3.fromRGB(150, 80, 255),
    accent2  = Color3.fromRGB(70, 190, 255),
    text     = Color3.fromRGB(240, 240, 250),
    sub      = Color3.fromRGB(155, 155, 180),
    on       = Color3.fromRGB(60, 225, 140),
    off      = Color3.fromRGB(75, 75, 92),
    danger   = Color3.fromRGB(255, 85, 95),
}

local function getGUIParent()
    if type(gethui) == "function" then
        local ok, hui = pcall(gethui)
        if ok and hui then return hui end
    end
    local ok, cg = pcall(function() return game:GetService("CoreGui") end)
    if ok and cg then return cg end
    return plr:WaitForChild("PlayerGui")
end

local function addCorner(inst, r)
    return create("UICorner", { CornerRadius = UDim.new(0, r or 8) }, inst)
end

local function addStroke(inst, color, thick, trans)
    return create("UIStroke", {
        Color = color or THEME.accent,
        Thickness = thick or 1,
        Transparency = trans or 0.4,
        ApplyStrokeMode = Enum.ApplyStrokeMode.Border,
    }, inst)
end

local function computeScale()
    local cam = Workspace.CurrentCamera
    local vp = (cam and cam.ViewportSize) or Vector2.new(1280, 720)
    local ux = (vp.X * 0.90) / CONFIG.UNIT_W
    local uy = (vp.Y * 0.84) / CONFIG.UNIT_H
    local s = math.min(ux, uy)
    if UIS.TouchEnabled and not UIS.MouseEnabled then
        s = math.min(s, 4.6)   -- mobile: jangan terlalu besar
    end
    return math.clamp(s, 2.4, 6.6)
end

local function buildUI()
    local S = computeScale()
    UI.S = S
    UI.W = math.floor(CONFIG.UNIT_W * S)
    UI.H = math.floor(CONFIG.UNIT_H * S)

    local gui = create("ScreenGui", {
        Name = "LIPZY_HUB",
        ResetOnSpawn = false,
        IgnoreGuiInset = true,
        DisplayOrder = 99999,
        ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
    })
    local parent = getGUIParent()
    local okParent = pcall(function() gui.Parent = parent end)
    if not okParent then
        gui.Parent = plr:WaitForChild("PlayerGui")
    end
    UI.gui = gui

    --================= WINDOW =================
    local win = create("Frame", {
        Name = "Window",
        AnchorPoint = Vector2.new(0.5, 0.5),
        Position = UDim2.new(0.5, 0, 0.5, 0),
        Size = UDim2.fromOffset(UI.W, UI.H),
        BackgroundColor3 = THEME.bg1,
        BorderSizePixel = 0,
        ClipsDescendants = true,
    }, gui)
    addCorner(win, 12)
    addStroke(win, THEME.accent, 1.4, 0.25)
    create("UIGradient", {
        Rotation = 90,
        Color = ColorSequence.new({
            ColorSequenceKeypoint.new(0, Color3.fromRGB(22, 22, 32)),
            ColorSequenceKeypoint.new(1, Color3.fromRGB(12, 12, 18)),
        }),
    }, win)
    UI.win = win

    --================= HEADER =================
    local headerH = math.floor(12 * S)
    local header = create("Frame", {
        Name = "Header",
        Size = UDim2.new(1, 0, 0, headerH),
        BackgroundColor3 = THEME.accent,
        BorderSizePixel = 0,
    }, win)
    create("UIGradient", {
        Rotation = 0,
        Color = ColorSequence.new({
            ColorSequenceKeypoint.new(0, THEME.accent),
            ColorSequenceKeypoint.new(1, THEME.accent2),
        }),
    }, header)
    UI.header = header

    local title = create("TextLabel", {
        Name = "Title",
        BackgroundTransparency = 1,
        Position = UDim2.new(0, math.floor(3 * S), 0, 0),
        Size = UDim2.new(1, -math.floor(16 * S), 1, 0),
        Text = CONFIG.TITLE,
        Font = Enum.Font.GothamBold,
        TextSize = math.floor(headerH * 0.46),
        TextColor3 = Color3.new(1, 1, 1),
        TextXAlignment = Enum.TextXAlignment.Left,
        TextTruncate = Enum.TextTruncate.AtEnd,
    }, header)

    local function headerBtn(text, xOffset, color, onClick)
        local btn = create("TextButton", {
            Name = "Btn" .. text,
            AnchorPoint = Vector2.new(1, 0.5),
            Position = UDim2.new(1, xOffset, 0.5, 0),
            Size = UDim2.fromOffset(math.floor(headerH * 0.62), math.floor(headerH * 0.62)),
            BackgroundColor3 = color,
            BackgroundTransparency = 0.25,
            Text = text,
            Font = Enum.Font.GothamBold,
            TextSize = math.floor(headerH * 0.3),
            TextColor3 = Color3.new(1, 1, 1),
            BorderSizePixel = 0,
            AutoButtonColor = true,
        }, header)
        addCorner(btn, math.floor(headerH * 0.2))
        btn.MouseButton1Click:Connect(function()
            pcall(onClick)
        end)
        return btn
    end

    local closeBtn = headerBtn("\u{2715}", -math.floor(1.5 * S), THEME.danger, function()
        unload()
    end)
    local minBtn = headerBtn("\u{2212}", -math.floor(5.0 * S), Color3.fromRGB(60, 60, 80), function()
        UI.minimized = not UI.minimized
        local speed = 0.18
        if UI.minimized then
            TweenService:Create(win, TweenInfo.new(speed), { Size = UDim2.fromOffset(UI.W, headerH) }):Play()
            UI.body.Visible = false
            UI.statusBar.Visible = false
        else
            TweenService:Create(win, TweenInfo.new(speed), { Size = UDim2.fromOffset(UI.W, UI.H) }):Play()
            UI.body.Visible = true
            UI.statusBar.Visible = true
        end
    end)

    --================= BODY (SCROLL) =================
    local statusH = math.floor(6 * S)
    local body = create("ScrollingFrame", {
        Name = "Body",
        Position = UDim2.new(0, 0, 0, headerH),
        Size = UDim2.new(1, 0, 1, -(headerH + statusH)),
        BackgroundTransparency = 1,
        BorderSizePixel = 0,
        ScrollBarThickness = math.floor(2 * S),
        ScrollBarImageColor3 = THEME.accent,
        CanvasSize = UDim2.new(0, 0, 0, 0),
        AutomaticCanvasSize = Enum.AutomaticSize.Y,
        ScrollingDirection = Enum.ScrollingDirection.Y,
        ElasticBehavior = Enum.ElasticBehavior.Never,
    }, win)
    UI.body = body
    create("UIListLayout", {
        Padding = UDim.new(0, math.floor(1.6 * S)),
        SortOrder = Enum.SortOrder.LayoutOrder,
    }, body)
    create("UIPadding", {
        PaddingTop = UDim.new(0, math.floor(1.6 * S)),
        PaddingBottom = UDim.new(0, math.floor(3 * S)),
        PaddingLeft = UDim.new(0, math.floor(2 * S)),
        PaddingRight = UDim.new(0, math.floor(2 * S)),
    }, body)

    --================= STATUS BAR =================
    local statusBar = create("Frame", {
        Name = "StatusBar",
        AnchorPoint = Vector2.new(0, 1),
        Position = UDim2.new(0, 0, 1, 0),
        Size = UDim2.new(1, 0, 0, statusH),
        BackgroundColor3 = THEME.bg2,
        BorderSizePixel = 0,
    }, win)
    create("UIGradient", {
        Rotation = 0,
        Color = ColorSequence.new({
            ColorSequenceKeypoint.new(0, Color3.fromRGB(20, 20, 30)),
            ColorSequenceKeypoint.new(1, Color3.fromRGB(30, 26, 46)),
        }),
    }, statusBar)
    UI.statusBar = statusBar

    StatusLabel = create("TextLabel", {
        Name = "Status",
        BackgroundTransparency = 1,
        Position = UDim2.new(0, math.floor(1.6 * S), 0, 0),
        Size = UDim2.new(0.72, 0, 1, 0),
        Text = "Ready",
        Font = Enum.Font.Gotham,
        TextSize = math.floor(statusH * 0.42),
        TextColor3 = THEME.sub,
        TextXAlignment = Enum.TextXAlignment.Left,
        TextTruncate = Enum.TextTruncate.AtEnd,
    }, statusBar)

    StatusFps = create("TextLabel", {
        Name = "Fps",
        BackgroundTransparency = 1,
        AnchorPoint = Vector2.new(1, 0),
        Position = UDim2.new(1, -math.floor(1.6 * S), 0, 0),
        Size = UDim2.new(0.3, 0, 1, 0),
        Text = "FPS: --",
        Font = Enum.Font.GothamBold,
        TextSize = math.floor(statusH * 0.42),
        TextColor3 = THEME.accent2,
        TextXAlignment = Enum.TextXAlignment.Right,
    }, statusBar)

    --================= HOOK RESIZE =================
    local function registerHook(fn)
        UI.hooks[#UI.hooks + 1] = fn
        pcall(fn)
    end

    --================= COMPONENT BUILDER =================
    local order = 0
    local function nextOrder()
        order = order + 1
        return order
    end

    local function addSection(text, color)
        local h = math.floor(7 * S)
        local f = create("Frame", {
            Name = "Section",
            Size = UDim2.new(1, 0, 0, h),
            BackgroundTransparency = 1,
            LayoutOrder = nextOrder(),
        }, body)
        local lbl = create("TextLabel", {
            BackgroundTransparency = 1,
            Position = UDim2.new(0, math.floor(0.6 * S), 0, 0),
            Size = UDim2.new(1, 0, 1, 0),
            Text = text,
            Font = Enum.Font.GothamBold,
            TextSize = math.floor(h * 0.5),
            TextColor3 = color or THEME.accent2,
            TextXAlignment = Enum.TextXAlignment.Left,
            TextTruncate = Enum.TextTruncate.AtEnd,
        }, f)
        registerHook(function()
            f.Size = UDim2.new(1, 0, 0, math.floor(7 * UI.S))
            lbl.TextSize = math.floor(7 * UI.S * 0.5)
        end)
        return f
    end

    local function addRowBase(hUnits)
        local h = math.floor(hUnits * S)
        local row = create("Frame", {
            Name = "Row",
            Size = UDim2.new(1, 0, 0, h),
            BackgroundColor3 = THEME.bg2,
            BackgroundTransparency = 0.15,
            BorderSizePixel = 0,
            LayoutOrder = nextOrder(),
        }, body)
        addCorner(row, math.floor(2 * S))
        addStroke(row, THEME.bg3, 1, 0.35)
        registerHook(function()
            row.Size = UDim2.new(1, 0, 0, math.floor(hUnits * UI.S))
        end)
        return row
    end

    --================= TOGGLE =================
    local function addToggle(text, default, callback)
        local row = addRowBase(9)
        local lbl = create("TextLabel", {
            Name = "Label",
            BackgroundTransparency = 1,
            Position = UDim2.new(0, math.floor(2 * S), 0, 0),
            Size = UDim2.new(1, -math.floor(13 * S), 1, 0),
            Text = text,
            Font = Enum.Font.GothamMedium,
            TextSize = math.floor(9 * S * 0.33),
            TextColor3 = THEME.text,
            TextXAlignment = Enum.TextXAlignment.Left,
            TextTruncate = Enum.TextTruncate.AtEnd,
        }, row)

        local track = create("Frame", {
            Name = "Track",
            AnchorPoint = Vector2.new(1, 0.5),
            Position = UDim2.new(1, -math.floor(2 * S), 0.5, 0),
            Size = UDim2.fromOffset(math.floor(6.6 * S), math.floor(3.4 * S)),
            BackgroundColor3 = default and THEME.on or THEME.off,
            BorderSizePixel = 0,
        }, row)
        addCorner(track, 999)

        local thumb = create("Frame", {
            Name = "Thumb",
            Position = UDim2.fromOffset(2, 2),
            Size = UDim2.fromOffset(math.floor(3.4 * S) - 4, math.floor(3.4 * S) - 4),
            BackgroundColor3 = Color3.new(1, 1, 1),
            BorderSizePixel = 0,
            ZIndex = 2,
        }, track)
        addCorner(thumb, 999)

        local obj = {
            value = default and true or false,
            row = row,
            label = lbl,
            track = track,
            thumb = thumb,
        }

        local function paint(animate)
            local on = obj.value
            local info = TweenInfo.new(animate and 0.16 or 0, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
            TweenService:Create(track, info, {
                BackgroundColor3 = on and THEME.on or THEME.off,
            }):Play()
            local trackW = math.floor(6.6 * UI.S)
            local pad = 2
            local thumbW = math.floor(3.4 * UI.S) - 4
            TweenService:Create(thumb, info, {
                Position = on and UDim2.fromOffset(trackW - thumbW - pad, pad) or UDim2.fromOffset(pad, pad),
            }):Play()
            TweenService:Create(lbl, info, {
                TextColor3 = on and Color3.new(1, 1, 1) or THEME.text,
            }):Play()
        end

        function obj:Set(v, silent)
            v = v and true or false
            if self.value == v then
                paint(true)
                return
            end
            self.value = v
            paint(true)
            if not silent and callback then
                local ok, err = pcall(callback, v)
                if not ok then
                    setStatus("Error: " .. tostring(err):sub(1, 40))
                end
            end
        end

        function obj:Toggle()
            self:Set(not self.value)
        end

        local function onInput(input)
            if input.UserInputType == Enum.UserInputType.Touch
                or input.UserInputType == Enum.UserInputType.MouseButton1 then
                obj:Toggle()
            end
        end
        row.InputBegan:Connect(onInput)
        track.InputBegan:Connect(onInput)
        thumb.InputBegan:Connect(onInput)

        registerHook(function()
            lbl.TextSize = math.floor(9 * UI.S * 0.33)
            lbl.Position = UDim2.new(0, math.floor(2 * UI.S), 0, 0)
            track.Size = UDim2.fromOffset(math.floor(6.6 * UI.S), math.floor(3.4 * UI.S))
            track.Position = UDim2.new(1, -math.floor(2 * UI.S), 0.5, 0)
            thumb.Size = UDim2.fromOffset(math.floor(3.4 * UI.S) - 4, math.floor(3.4 * UI.S) - 4)
            paint(false)
        end)

        UI.toggles[#UI.toggles + 1] = obj
        paint(false)
        return obj
    end

    --================= SLIDER =================
    local function addSlider(text, minV, maxV, default, callback)
        local row = addRowBase(8)
        local lbl = create("TextLabel", {
            Name = "Label",
            BackgroundTransparency = 1,
            Position = UDim2.new(0, math.floor(2 * S), 0, 0),
            Size = UDim2.new(0.6, 0, 0.5, 0),
            Text = text,
            Font = Enum.Font.GothamMedium,
            TextSize = math.floor(8 * S * 0.32),
            TextColor3 = THEME.text,
            TextXAlignment = Enum.TextXAlignment.Left,
            TextTruncate = Enum.TextTruncate.AtEnd,
        }, row)

        local valLbl = create("TextLabel", {
            Name = "Value",
            BackgroundTransparency = 1,
            AnchorPoint = Vector2.new(1, 0),
            Position = UDim2.new(1, -math.floor(2 * S), 0, 0),
            Size = UDim2.new(0.38, 0, 0.5, 0),
            Text = tostring(default),
            Font = Enum.Font.GothamBold,
            TextSize = math.floor(8 * S * 0.32),
            TextColor3 = THEME.accent2,
            TextXAlignment = Enum.TextXAlignment.Right,
        }, row)

        local track = create("Frame", {
            Name = "Track",
            AnchorPoint = Vector2.new(0, 1),
            Position = UDim2.new(0, math.floor(2 * S), 1, -math.floor(1.2 * S)),
            Size = UDim2.new(1, -math.floor(4 * S), 0, math.max(6, math.floor(1.1 * S))),
            BackgroundColor3 = THEME.off,
            BorderSizePixel = 0,
        }, row)
        addCorner(track, 999)

        local fill = create("Frame", {
            Name = "Fill",
            Size = UDim2.new(0, 0, 1, 0),
            BackgroundColor3 = THEME.accent,
            BorderSizePixel = 0,
        }, track)
        addCorner(fill, 999)

        local obj = { value = default, min = minV, max = maxV, track = track, fill = fill }

        local function ratio()
            return math.clamp((obj.value - obj.min) / math.max(1, (obj.max - obj.min)), 0, 1)
        end

        local function paint()
            TweenService:Create(fill, TweenInfo.new(0.08), { Size = UDim2.new(ratio(), 0, 1, 0) }):Play()
            valLbl.Text = tostring(math.floor(obj.value))
        end

        local function setFromX(x)
            local absX = track.AbsolutePosition.X
            local w = track.AbsoluteSize.X
            if w <= 0 then return end
            local a = math.clamp((x - absX) / w, 0, 1)
            obj.value = math.floor(obj.min + a * (obj.max - obj.min))
            paint()
            if callback then
                local ok, err = pcall(callback, obj.value)
                if not ok then
                    setStatus("Error: " .. tostring(err):sub(1, 40))
                end
            end
        end

        local draggingSlider = false
        track.InputBegan:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.Touch
                or input.UserInputType == Enum.UserInputType.MouseButton1 then
                draggingSlider = true
                setFromX(input.Position.X)
            end
        end)
        bind(UIS.InputChanged, function(input)
            if not draggingSlider then return end
            if input.UserInputType == Enum.UserInputType.Touch
                or input.UserInputType == Enum.UserInputType.MouseMovement then
                setFromX(input.Position.X)
            end
        end)
        bind(UIS.InputEnded, function(input)
            if input.UserInputType == Enum.UserInputType.Touch
                or input.UserInputType == Enum.UserInputType.MouseButton1 then
                draggingSlider = false
            end
        end)

        function obj:Set(v)
            obj.value = math.clamp(v, obj.min, obj.max)
            paint()
        end

        registerHook(function()
            lbl.TextSize = math.floor(8 * UI.S * 0.32)
            valLbl.TextSize = math.floor(8 * UI.S * 0.32)
            track.Size = UDim2.new(1, -math.floor(4 * UI.S), 0, math.max(6, math.floor(1.1 * UI.S)))
            paint()
        end)

        UI.sliders[#UI.sliders + 1] = obj
        paint()
        return obj
    end

    --================= BUTTON =================
    local function addButton(text, color, callback)
        local row = addRowBase(8)
        local btn = create("TextButton", {
            Name = "Button",
            Size = UDim2.new(1, 0, 1, 0),
            BackgroundColor3 = color or THEME.accent,
            BackgroundTransparency = 0.15,
            Text = text,
            Font = Enum.Font.GothamBold,
            TextSize = math.floor(8 * S * 0.34),
            TextColor3 = Color3.new(1, 1, 1),
            BorderSizePixel = 0,
            AutoButtonColor = true,
        }, row)
        addCorner(btn, math.floor(2 * S))
        btn.MouseButton1Click:Connect(function()
            local ok, err = pcall(callback)
            if not ok then
                setStatus("Error: " .. tostring(err):sub(1, 40))
            end
        end)
        registerHook(function()
            btn.TextSize = math.floor(8 * UI.S * 0.34)
        end)
        return btn
    end

    UI.addSection = addSection
    UI.addToggle = addToggle
    UI.addSlider = addSlider
    UI.addButton = addButton
    UI.registerHook = registerHook

    --================= DRAG WINDOW (mouse + touch) =================
    local dragging = false
    local dragStart, startPos
    header.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
            or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = win.Position
        end
    end)
    bind(UIS.InputChanged, function(input)
        if not dragging then return end
        if input.UserInputType == Enum.UserInputType.MouseMovement
            or input.UserInputType == Enum.UserInputType.Touch then
            local delta = input.Position - dragStart
            local cam = Workspace.CurrentCamera
            local vp = (cam and cam.ViewportSize) or Vector2.new(1280, 720)
            local maxX = math.max(0, (vp.X - UI.W) * 0.5)
            local maxY = math.max(0, (vp.Y - UI.H) * 0.5)
            local nx = math.clamp(startPos.X.Offset + delta.X, -maxX, maxX)
            local ny = math.clamp(startPos.Y.Offset + delta.Y, -maxY, maxY)
            win.Position = UDim2.new(startPos.X.Scale, nx, startPos.Y.Scale, ny)
        end
    end)
    bind(UIS.InputEnded, function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
            or input.UserInputType == Enum.UserInputType.Touch then
            dragging = false
        end
    end)

    --================= RESPONSIVE (viewport berubah) =================
    local cam = Workspace.CurrentCamera
    if cam then
        bind(cam:GetPropertyChangedSignal("ViewportSize"), function()
            task.delay(0.25, function()
                local newS = computeScale()
                if math.abs(newS - UI.S) < 0.15 then return end
                UI.S = newS
                UI.W = math.floor(CONFIG.UNIT_W * newS)
                UI.H = math.floor(CONFIG.UNIT_H * newS)
                pcall(function()
                    header.Size = UDim2.new(1, 0, 0, math.floor(12 * newS))
                    title.TextSize = math.floor(12 * newS * 0.46)
                    closeBtn.Size = UDim2.fromOffset(math.floor(12 * newS * 0.62), math.floor(12 * newS * 0.62))
                    minBtn.Size = closeBtn.Size
                    closeBtn.Position = UDim2.new(1, -math.floor(1.5 * newS), 0.5, 0)
                    minBtn.Position = UDim2.new(1, -math.floor(5.0 * newS), 0.5, 0)
                    if not UI.minimized then
                        win.Size = UDim2.fromOffset(UI.W, UI.H)
                    else
                        win.Size = UDim2.fromOffset(UI.W, math.floor(12 * newS))
                    end
                    body.Position = UDim2.new(0, 0, 0, math.floor(12 * newS))
                    body.Size = UDim2.new(1, 0, 1, -(math.floor(12 * newS) + math.floor(6 * newS)))
                    statusBar.Size = UDim2.new(1, 0, 0, math.floor(6 * newS))
                    StatusLabel.TextSize = math.floor(6 * newS * 0.42)
                    StatusFps.TextSize = math.floor(6 * newS * 0.42)
                    for i = 1, #UI.hooks do
                        pcall(UI.hooks[i])
                    end
                end)
            end)
        end)
    end

    return UI
end

--===================================================================
-- 16. WIRING FITUR -> UI
--===================================================================
local function refreshZoneStatus()
    if SafeZone.manual then
        setStatus("Zona aman: MANUAL (posisi ditandai)")
        return
    end
    local z = SafeZone.part
    if z and z.Parent then
        setStatus("Zona aman: " .. z.Name)
    else
        z = findSafeZone()
        if z then
            setStatus("Zona aman: " .. z.Name)
        else
            setStatus("Zona aman belum ditemukan")
        end
    end
end

local ResetSkip = {}

local function wireUI()
    local UI_ = buildUI()

    --================ FITUR 1 ================
    UI_.addSection("FITUR 1 - ANTI GUARD + INSTANT STEAL", THEME.accent2)

    UI_.addToggle("INSTANT STEAL + AUTO DELIVER", false, function(v)
        State.autoSteal = v
        State.instantPick = v or State.instantPick
        if v then
            if not SafeZone.part then findSafeZone() end
            startStealLoop()
            notify("LIPZY HUB", "Auto Steal + Auto Deliver AKTIF", 3)
        else
            stopStealLoop()
        end
    end)

    UI_.addToggle("INSTANT PICK (1x Tap / Hold = 0)", false, function(v)
        State.instantPick = v
        if v then
            local n = scanPrompts(true)
            setStatus("Instant Pick ON - " .. tostring(n) .. " prompt diproses")
            notify("LIPZY HUB", "Instant Pick ON (" .. tostring(#PromptPool) .. " prompt)", 3)
        else
            if not State.autoSteal then
                cleanupPrompts()
                setStatus("Instant Pick OFF")
            end
        end
    end)

    UI_.addToggle("GODMODE / ANTI-RESET / ANTI-HIT", false, function(v)
        State.godmode = v
        if v then
            startGodmode()
            notify("LIPZY HUB", "Godmode + Anti Reset ON", 3)
        else
            stopGodmode()
        end
    end)

    UI_.addToggle("SUPER SPEED (" .. tostring(CONFIG.WALK_DEFAULT) .. ")", false, function(v)
        State.superSpeed = v
        applyWalkSpeed()
        setStatus("Super Speed " .. (v and tostring(State.walkSpeed) or "OFF"))
    end)

    -- toggle yang TIDAK ikut di-reset oleh tombol "STOP SEMUA FITUR"
    ResetSkip[#ResetSkip + 1] = UI_.addToggle("FLY MODE (ANTI-BUG CARRY)", true, function(v)
        State.flyCarry = v
        if not v then Fly:Stop() end
        setStatus("Fly Mode " .. (v and "ON (velocity)" or "OFF (mikro-step)"))
    end)

    UI_.addSlider("Walk Speed", 100, 1000, CONFIG.WALK_DEFAULT, function(v)
        State.walkSpeed = v
        if State.superSpeed then
            local hum = getHum()
            if hum then pcall(function() hum.WalkSpeed = v end) end
        end
    end)

    UI_.addSlider("Fly Speed", 150, 1500, CONFIG.FLY_DEFAULT, function(v)
        State.flySpeed = v
    end)

    UI_.addSlider("Egg Range", 100, 1500, CONFIG.PICK_RANGE, function(v)
        State.pickRange = v
    end)

    --================ FITUR 2 ================
    UI_.addSection("FITUR 2 - NO GUARD AGGRO (15 ZONA)", Color3.fromRGB(255, 170, 60))

    local guardToggle
    guardToggle = UI_.addToggle("NO GUARD AGGRO (FREEZE PENJAGA)", false, function(v)
        State.antiGuard = v
        if v then
            startGuardLoop()
            notify("LIPZY HUB", "No Guard Aggro ON - penjaga dibekukan di posisi aslinya", 3)
        else
            stopGuardLoop()
            notify("LIPZY HUB", "No Guard Aggro OFF - penjaga dilepas", 3)
        end
    end)

    ResetSkip[#ResetSkip + 1] = UI_.addToggle("HARD FREEZE (KUNCI POSISI ASLI)", true, function(v)
        State.hardFreeze = v
        setStatus("Hard Freeze " .. (v and "ON" or "OFF"))
    end)

    UI_.addButton("Rescan Penjaga Sekarang", Color3.fromRGB(255, 140, 60), function()
        if not State.antiGuard then
            guardToggle:Set(true, true)
            State.antiGuard = true
            startGuardLoop()
        end
        local before = 0
        for _ in pairs(Guards.frozen) do before = before + 1 end
        local found = 0
        pcall(function() found = scanGuards() end)
        setStatus("Penjaga dibekukan: " .. tostring(before + found) .. " (15 zona)")
        notify("LIPZY HUB", "Scan penjaga selesai - " .. tostring(before + found) .. " dibekukan", 4)
    end)

    --================ FITUR 3 ================
    UI_.addSection("FITUR 3 - FPS UNLOCKER 120", Color3.fromRGB(80, 230, 160))

    UI_.addToggle("FPS UNLOCKER " .. tostring(CONFIG.FPS_DEFAULT), false, function(v)
        State.fpsUnlock = v
        if v then
            enableFps()
            notify("LIPZY HUB", "FPS Unlocker ON -> " .. tostring(State.fpsTarget) .. " FPS", 3)
        else
            disableFps()
        end
    end)

    UI_.addSlider("FPS Target", 30, 240, CONFIG.FPS_DEFAULT, function(v)
        State.fpsTarget = v
        if State.fpsUnlock then
            setFpsCap(v)
            setStatus("FPS target: " .. tostring(v))
        end
    end)

    UI_.addToggle("SMOOTH EKSTRA (EFEK & BAYANGAN OFF)", false, function(v)
        State.smoothExtra = v
        applySmoothExtra(v)
        setStatus("Smooth Extra " .. (v and "ON" or "OFF"))
    end)

    --================ TOOLS ================
    UI_.addSection("ZONA AMAN & TOOLS", Color3.fromRGB(200, 200, 230))

    UI_.addButton("Rescan Zona Aman", THEME.accent, function()
        SafeZone.manual = nil
        local z, sc = findSafeZone()
        if z then
            notify("LIPZY HUB", "Zona aman ditemukan: " .. z.Name .. " (skor " .. tostring(sc) .. ")", 4)
        else
            notify("LIPZY HUB", "Zona aman TIDAK ditemukan. Berdiri di zona aman lalu klik 'Set Zona Aman di Sini'.", 6)
        end
        refreshZoneStatus()
    end)

    UI_.addButton("Set Zona Aman di Sini (Manual)", THEME.accent2, function()
        local root = getRoot()
        if not root then
            notify("LIPZY HUB", "Character belum spawn!", 4)
            return
        end
        local marker = create("Part", {
            Name = "LIPZY_SAFEZONE_MARKER",
            Anchored = true,
            CanCollide = false,
            Transparency = 1,
            Size = Vector3.new(24, 18, 24),
            Position = root.Position,
            Parent = Workspace,
        })
        SafeZone.manual = marker
        SafeZone.part = marker
        notify("LIPZY HUB", "Zona aman manual di-set di posisi kamu sekarang.", 4)
        setStatus("Zona aman: MANUAL")
    end)

    UI_.addButton("STOP SEMUA FITUR", Color3.fromRGB(255, 120, 60), function()
        State.autoSteal = false
        State.instantPick = false
        State.antiGuard = false
        State.superSpeed = false
        State.fpsUnlock = false
        State.smoothExtra = false
        State.godmode = false
        stopStealLoop()
        stopGuardLoop()
        cleanupPrompts()
        disableFps()
        applySmoothExtra(false)
        stopGodmode()
        applyWalkSpeed()
        for i = 1, #UI_.toggles do
            local t = UI_.toggles[i]
            local skip = false
            for s = 1, #ResetSkip do
                if ResetSkip[s] == t then
                    skip = true
                    break
                end
            end
            if not skip then
                pcall(function() t:Set(false, true) end)
            end
        end
        setStatus("Semua fitur OFF")
        notify("LIPZY HUB", "Semua fitur dimatikan & di-restore.", 4)
    end)

    UI_.addButton("UNLOAD / HAPUS GUI", Color3.fromRGB(190, 40, 55), function()
        unload()
    end)

    refreshZoneStatus()
    return UI_
end

--===================================================================
-- 17. LOOP RINGAN: FPS COUNTER, AUTO RE-APPLY, PROMPT BARU
--===================================================================
local function startBackgroundLoop()
    -- FPS counter
    local acc, frames = 0, 0
    bind(RunService.RenderStepped, function(dt)
        acc = acc + dt
        frames = frames + 1
        if acc >= 0.5 then
            local fps = math.floor(frames / acc)
            if StatusFps then
                pcall(function() StatusFps.Text = "FPS: " .. tostring(fps) end)
            end
            acc, frames = 0, 0
        end
    end)

    -- re-apply WalkSpeed + instant prompt + godmode saat respawn
    bind(plr.CharacterAdded, function(char)
        Speed.savedKey = nil
        Speed.savedValue = nil
        task.wait(0.6)
        if State.godmode then applyGodmode() end
        if State.superSpeed then applyWalkSpeed() end
        if State.autoSteal or State.instantPick then pcall(function() scanPrompts(true) end) end
    end)

    -- prompt baru (telur respawn)
    bind(Workspace.DescendantAdded, function(inst)
        if inst:IsA("ProximityPrompt") then
            if State.instantPick or State.autoSteal then
                if (not State.filterPrompt) or isEggPrompt(inst) then
                    applyPrompt(inst)
                end
            end
        elseif State.fpsUnlock and isEffect(inst) then
            if not Perf.effects[inst] then
                local ok, v = pcall(function() return inst.Enabled end)
                if ok then
                    Perf.effects[inst] = v
                    pcall(function() inst.Enabled = false end)
                end
            end
        elseif State.antiGuard and inst:IsA("Humanoid") and inst.Health > 0 then
            if not Guards.frozen[inst] and isGuardHumanoid(inst) then
                freezeGuard(inst)
            end
        end
    end)

    -- loop ringan: refresh prompt & jaga super speed (TIDAK melakukan scan berat)
    task.spawn(function()
        while true do
            task.wait(0.5)
            if State.instantPick or State.autoSteal then
                pcall(reapplyPrompts)
            end
            if State.superSpeed then
                local hum = getHum()
                if hum and math.abs(hum.WalkSpeed - State.walkSpeed) > 1 then
                    pcall(function() hum.WalkSpeed = State.walkSpeed end)
                end
            end
        end
    end)
end

--===================================================================
-- 18. UNLOAD (restore semua & hapus GUI)
--===================================================================
function unload()
    State.autoSteal = false
    State.instantPick = false
    State.antiGuard = false
    State.godmode = false
    State.fpsUnlock = false
    State.superSpeed = false
    State.smoothExtra = false

    pcall(stopStealLoop)
    pcall(stopGuardLoop)
    pcall(cleanupPrompts)
    pcall(stopGodmode)
    pcall(disableFps)
    pcall(applySmoothExtra, false)
    pcall(function()
        local hum = getHum()
        if hum and Speed.savedValue then
            hum.WalkSpeed = Speed.savedValue
        end
    end)
    pcall(Fly.Stop)

    if SafeZone.manual then
        pcall(function() SafeZone.manual:Destroy() end)
        SafeZone.manual = nil
    end

    for i = #Cleanups, 1, -1 do
        pcall(Cleanups[i])
    end
    Cleanups = {}

    for i = 1, #Connections do
        pcall(function() Connections[i]:Disconnect() end)
    end
    Connections = {}

    if UI.gui then
        pcall(function() UI.gui:Destroy() end)
        UI.gui = nil
    end
    if GENV and type(GENV) == "table" then
        GENV.LIPZY_HUB = nil
    end
    print("[LIPZY HUB] Unloaded. Semua perubahan dikembalikan.")
end

GENV.LIPZY_HUB = {
    Unload = unload,
    State = State,
    Stats = Stats,
    CONFIG = CONFIG,
    UI = UI,
    Fly = Fly,
    Guards = Guards,
    Perf = Perf,
}

--===================================================================
-- 19. START
--===================================================================
local okStart, errStart = pcall(function()
    wireUI()
    startBackgroundLoop()
end)

if not okStart then
    warn("[LIPZY HUB] Gagal membangun UI: " .. tostring(errStart))
    notify("LIPZY HUB", "Gagal membangun UI: " .. tostring(errStart):sub(1, 60), 8)
    return
end

notify("LIPZY HUB", "Loaded! Aktifkan toggle untuk mulai (Mobile friendly).", 5)
setStatus("Ready - aktifkan fitur yang diinginkan")
print("[LIPZY HUB] Steal an Egg loaded. Toggle semua fitur OFF secara default (aman).")
