--[[
===============================================================
 TRANSFORMER ENGINE — AUTÔNOMO / MOBILE FIRST
 Um único LocalScript. Não exige modelos, animações, sons,
 pastas, RemoteEvents, módulos ou IDs preparados pelo criador.

 INSTALAÇÃO:
 StarterPlayer > StarterPlayerScripts > LocalScript
 Cole este arquivo inteiro.

 O script constrói um mecha e um caminhão usando Parts nativas,
 esconde/restaura localmente o avatar, anima as placas por CFrame,
 e cria UI, partículas, luzes e controles mobile em tempo de execução.

 LIMITAÇÕES IMPORTANTES:
 • Os modelos gerados são mecânicos/estilizados, não malhas 3D
   cinematográficas modeladas à mão.
 • É uma transformação visual local. Um LocalScript sozinho não
   replica o mecha para os outros jogadores nem controla autoridade
   física do servidor.
 • O Roblox não fornece síntese de efeitos sonoros arbitrários em
   Luau puro sem fonte de áudio; esta versão não finge possuir SFX.
===============================================================
]]

---------------------------
-- 1. SERVIÇOS E CONFIGURAÇÃO
---------------------------
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Lighting = game:GetService("Lighting")
local Workspace = game:GetService("Workspace")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

local CFG = {
    TRANSFORM_TIME = 3.15,
    HUMANOID_SPEED = 16,
    MECHA_SPEED = 27,
    VEHICLE_SPEED = 58,
    MECHA_JUMP_POWER = 58,
    MECHA_JUMP_HEIGHT = 8,
    WALK_SWAY = 0.52,
    RUN_SWAY = 0.72,
    MECHA_FOV = 77,
    VEHICLE_FOV = 80,
    HUMANOID_FOV = 70,
    FX_PARTICLE_RATE = 16,
    FX_LOW_PARTICLE_RATE = 8,
    SHOW_KEYBOARD_SHORTCUTS = true,
}

local COLORS = {
    Red = Color3.fromRGB(190, 39, 43),
    RedLight = Color3.fromRGB(255, 75, 61),
    Blue = Color3.fromRGB(28, 93, 190),
    BlueLight = Color3.fromRGB(0, 190, 255),
    BlueGlass = Color3.fromRGB(20, 70, 112),
    Steel = Color3.fromRGB(87, 105, 122),
    Silver = Color3.fromRGB(170, 186, 198),
    Dark = Color3.fromRGB(20, 28, 39),
    Black = Color3.fromRGB(7, 10, 16),
    Gold = Color3.fromRGB(255, 181, 48),
    White = Color3.fromRGB(225, 240, 255),
    Tire = Color3.fromRGB(14, 17, 22),
    Glass = Color3.fromRGB(9, 27, 44),
}

local FORMS = {
    HUMANOIDE = "HUMANOIDE",
    MECHA = "MECHA",
    VEICULO = "VEÍCULO",
}

local form = FORMS.HUMANOIDE
local transition = nil
local character = nil
local humanoid = nil
local hrp = nil
local avatarParts = {}
local descendantConnection = nil
local normalProperties = nil
local camera = Workspace.CurrentCamera
local initialFOV = camera and camera.FieldOfView or CFG.HUMANOID_FOV
CFG.HUMANOID_FOV = initialFOV
local lastRootCF = nil
local destroyed = false
local walkPhase = 0
local wheelPhase = 0
local effectTick = 0

---------------------------
-- 2. FUNÇÕES UTILITÁRIAS
---------------------------
local function safeCall(fn)
    local ok, result = pcall(fn)
    if not ok then
        warn("[Transformer Engine] " .. tostring(result))
    end
    return ok, result
end

local function clamp01(value)
    return math.clamp(value, 0, 1)
end

local function smoothstep(value)
    value = clamp01(value)
    return value * value * (3 - 2 * value)
end

local function angleCF(x, y, z)
    return CFrame.Angles(math.rad(x or 0), math.rad(y or 0), math.rad(z or 0))
end

local function setPartBasics(part, name, size, color, material, transparency)
    part.Name = name
    part.Size = size
    part.Color = color or COLORS.Steel
    part.Material = material or Enum.Material.Metal
    part.Transparency = transparency or 0
    part.Anchored = true
    part.CanCollide = false
    part.CanTouch = false
    part.CanQuery = false
    part.CastShadow = part.Material ~= Enum.Material.Neon
    part.TopSurface = Enum.SurfaceType.Smooth
    part.BottomSurface = Enum.SurfaceType.Smooth
end

local function createVisualModel(name)
    local old = Workspace:FindFirstChild(name)
    if old then
        old:Destroy()
    end

    local model = Instance.new("Model")
    model.Name = name
    model.Parent = Workspace

    local root = Instance.new("Part")
    setPartBasics(root, "VisualRoot", Vector3.new(0.2, 0.2, 0.2), COLORS.Black, Enum.Material.SmoothPlastic, 1)
    root.CFrame = CFrame.new(0, -1000, 0)
    root.Parent = model
    model.PrimaryPart = root

    return model
end

local function newPart(model, list, name, size, cf, color, material, shape, group, pivot, transparency, className)
    local part
    if className == "WedgePart" then
        part = Instance.new("WedgePart")
    else
        part = Instance.new("Part")
        if shape then
            part.Shape = shape
        end
    end

    setPartBasics(part, name, size, color, material, 1)
    part.CFrame = cf
    part.Parent = model

    local index = #list + 1
    -- Dispersão determinística: não usa math.random dentro do loop de renderização.
    local scatterX = math.sin(index * 12.37 + #name) * (1.6 + (index % 3) * 0.55)
    local scatterY = math.cos(index * 8.13 + #name) * (1.2 + (index % 4) * 0.40)
    local scatterZ = math.sin(index * 5.71 + #name * 0.7) * (1.6 + (index % 5) * 0.35)
    local scatter = CFrame.new(scatterX, scatterY, scatterZ)
        * CFrame.Angles(math.rad((index * 31) % 130 - 65), math.rad((index * 47) % 180 - 90), math.rad((index * 23) % 100 - 50))

    local item = {
        part = part,
        name = name,
        baseCF = cf,
        baseTransparency = transparency or 0,
        group = group or "body",
        pivot = pivot or Vector3.zero,
        scatter = scatter,
        index = index,
    }
    table.insert(list, item)
    return part, item
end

local function addSparkEmitter(part, colorSequence, rate)
    local attachment = Instance.new("Attachment")
    attachment.Name = "GeneratedSparkAttachment"
    attachment.Parent = part

    local emitter = Instance.new("ParticleEmitter")
    emitter.Name = "GeneratedSparks"
    emitter.Texture = "rbxasset://textures/particles/sparkles_main.dds"
    emitter.Color = colorSequence or ColorSequence.new(COLORS.BlueLight, COLORS.Gold)
    emitter.LightEmission = 0.85
    emitter.LightInfluence = 0
    emitter.Lifetime = NumberRange.new(0.18, 0.55)
    emitter.Speed = NumberRange.new(3, 9)
    emitter.Rate = rate or CFG.FX_LOW_PARTICLE_RATE
    emitter.SpreadAngle = Vector2.new(180, 180)
    emitter.Rotation = NumberRange.new(0, 360)
    emitter.RotSpeed = NumberRange.new(-180, 180)
    emitter.Size = NumberSequence.new({
        NumberSequenceKeypoint.new(0, 0.19),
        NumberSequenceKeypoint.new(0.55, 0.10),
        NumberSequenceKeypoint.new(1, 0),
    })
    emitter.Transparency = NumberSequence.new({
        NumberSequenceKeypoint.new(0, 0.05),
        NumberSequenceKeypoint.new(1, 1),
    })
    emitter.Enabled = false
    emitter.Parent = attachment
    return emitter
end

local function addSmokeEmitter(part)
    local attachment = Instance.new("Attachment")
    attachment.Name = "GeneratedSmokeAttachment"
    attachment.Position = Vector3.new(0, -0.25, 0)
    attachment.Parent = part

    local emitter = Instance.new("ParticleEmitter")
    emitter.Name = "GeneratedLandingSmoke"
    emitter.Texture = "rbxasset://textures/particles/smoke_main.dds"
    emitter.Color = ColorSequence.new(Color3.fromRGB(95, 108, 122), Color3.fromRGB(30, 36, 44))
    emitter.LightInfluence = 0.2
    emitter.Lifetime = NumberRange.new(0.45, 1.15)
    emitter.Speed = NumberRange.new(2, 5)
    emitter.Rate = 0
    emitter.SpreadAngle = Vector2.new(35, 35)
    emitter.Size = NumberSequence.new({
        NumberSequenceKeypoint.new(0, 0.6),
        NumberSequenceKeypoint.new(1, 2.1),
    })
    emitter.Transparency = NumberSequence.new({
        NumberSequenceKeypoint.new(0, 0.25),
        NumberSequenceKeypoint.new(1, 1),
    })
    emitter.Enabled = false
    emitter.Parent = attachment
    return emitter
end

local function createTrail(part, color)
    local a0 = Instance.new("Attachment")
    a0.Name = "TrailAttachmentA"
    a0.Position = Vector3.new(0, 0.2, -0.35)
    a0.Parent = part

    local a1 = Instance.new("Attachment")
    a1.Name = "TrailAttachmentB"
    a1.Position = Vector3.new(0, -0.2, -0.35)
    a1.Parent = part

    local trail = Instance.new("Trail")
    trail.Name = "GeneratedMotionTrail"
    trail.Attachment0 = a0
    trail.Attachment1 = a1
    trail.Color = ColorSequence.new(color or COLORS.BlueLight)
    trail.LightEmission = 0.9
    trail.Lifetime = 0.16
    trail.MinLength = 0.05
    trail.Transparency = NumberSequence.new({
        NumberSequenceKeypoint.new(0, 0.1),
        NumberSequenceKeypoint.new(1, 1),
    })
    trail.Enabled = false
    trail.Parent = part
    return trail
end

local function buildPointLight(parent, color, range, brightness)
    local light = Instance.new("PointLight")
    light.Color = color
    light.Range = range
    light.Brightness = brightness
    light.Shadows = false
    light.Parent = parent
    return light
end

---------------------------
-- 3. CONSTRUÇÃO PROCEDURAL DO MECHA
---------------------------
local mechaModel = createVisualModel("__TransformerEngine_Mecha_" .. player.UserId)
local mechaParts = {}
local mechaEmitters = {}
local mechaTrails = {}
local mechaCoreLight = nil
local mechaEyeParts = {}
local mechaSmokeEmitters = {}

local function buildMecha()
    local m = mechaModel
    local parts = mechaParts

    local function B(name, size, x, y, z, color, material, group, pivot, rx, ry, rz, shape, transparency, className)
        local cf = CFrame.new(x, y, z) * angleCF(rx, ry, rz)
        local p = newPart(m, parts, name, size, cf, color, material, shape, group, pivot, transparency, className)
        return p
    end

    local BODY = Vector3.zero
    local LSH = Vector3.new(-1.48, 2.30, 0)
    local RSH = Vector3.new(1.48, 2.30, 0)
    local LHIP = Vector3.new(-0.65, -0.68, 0)
    local RHIP = Vector3.new(0.65, -0.68, 0)
    local HEAD = Vector3.new(0, 3.85, 0)

    -- Coluna central, torso e placas abdominais.
    B("Torso_Core", Vector3.new(2.05, 2.15, 1.05), 0, 1.95, 0.05, COLORS.Dark, Enum.Material.Metal)
    B("Chest_Outer_Armor", Vector3.new(2.60, 1.28, 1.18), 0, 2.72, -0.03, COLORS.Blue, Enum.Material.Metal)
    B("Chest_Center_Plate", Vector3.new(0.72, 1.03, 0.18), 0, 2.63, -0.69, COLORS.Silver, Enum.Material.DiamondPlate)
    B("Chest_Left_Red_Panel", Vector3.new(0.77, 1.05, 0.22), -0.82, 2.70, -0.69, COLORS.Red, Enum.Material.Metal, "body", BODY, 0, 0, -7)
    B("Chest_Right_Red_Panel", Vector3.new(0.77, 1.05, 0.22), 0.82, 2.70, -0.69, COLORS.Red, Enum.Material.Metal, "body", BODY, 0, 0, 7)
    B("Windshield_Left", Vector3.new(0.47, 0.72, 0.09), -0.82, 2.78, -0.83, COLORS.BlueGlass, Enum.Material.Glass, "body", BODY, 0, 0, -8)
    B("Windshield_Right", Vector3.new(0.47, 0.72, 0.09), 0.82, 2.78, -0.83, COLORS.BlueGlass, Enum.Material.Glass, "body", BODY, 0, 0, 8)
    B("Chest_Upper_Brow", Vector3.new(2.35, 0.20, 0.20), 0, 3.36, -0.62, COLORS.Silver, Enum.Material.Metal)
    B("Abdominal_Plate_01", Vector3.new(1.55, 0.34, 0.83), 0, 1.27, -0.52, COLORS.Steel, Enum.Material.DiamondPlate)
    B("Abdominal_Plate_02", Vector3.new(1.38, 0.28, 0.78), 0, 0.91, -0.55, COLORS.Dark, Enum.Material.Metal)
    B("Abdominal_Plate_03", Vector3.new(1.18, 0.22, 0.70), 0, 0.62, -0.55, COLORS.Silver, Enum.Material.Metal)
    B("Waist_Belt", Vector3.new(1.92, 0.35, 1.03), 0, 0.39, 0.02, COLORS.Blue, Enum.Material.Metal)
    B("Belt_Core", Vector3.new(0.38, 0.25, 0.10), 0, 0.39, -0.57, COLORS.Gold, Enum.Material.Neon)

    local core = B("Energon_Core", Vector3.new(0.50, 0.50, 0.23), 0, 2.52, -0.91, COLORS.BlueLight, Enum.Material.Neon, "body", BODY, 0, 0, 0, Enum.PartType.Ball)
    mechaCoreLight = buildPointLight(core, COLORS.BlueLight, 13, 2.3)
    table.insert(mechaEmitters, addSparkEmitter(core, ColorSequence.new(COLORS.BlueLight, COLORS.White), 2))

    -- Costas e propulsores.
    B("Backpack_Main", Vector3.new(1.75, 1.90, 0.72), 0, 2.02, 0.82, COLORS.Dark, Enum.Material.Metal)
    B("Backpack_Left_Armor", Vector3.new(0.63, 1.45, 0.65), -0.67, 2.12, 1.04, COLORS.Blue, Enum.Material.Metal)
    B("Backpack_Right_Armor", Vector3.new(0.63, 1.45, 0.65), 0.67, 2.12, 1.04, COLORS.Blue, Enum.Material.Metal)
    B("Thruster_Left", Vector3.new(0.42, 1.15, 0.48), -0.67, 1.37, 1.13, COLORS.Steel, Enum.Material.Metal)
    B("Thruster_Right", Vector3.new(0.42, 1.15, 0.48), 0.67, 1.37, 1.13, COLORS.Steel, Enum.Material.Metal)
    B("Thruster_Left_Glow", Vector3.new(0.24, 0.15, 0.26), -0.67, 0.77, 1.13, COLORS.BlueLight, Enum.Material.Neon)
    B("Thruster_Right_Glow", Vector3.new(0.24, 0.15, 0.26), 0.67, 0.77, 1.13, COLORS.BlueLight, Enum.Material.Neon)
    B("Back_Spine", Vector3.new(0.36, 1.45, 0.28), 0, 1.95, 1.28, COLORS.Silver, Enum.Material.Metal)

    -- Cabeça: capacete, máscara, olhos e antenas.
    B("Head_Helmet", Vector3.new(0.88, 0.86, 0.74), 0, 3.98, 0, COLORS.Blue, Enum.Material.Metal, "head", HEAD)
    B("Head_Crown", Vector3.new(0.65, 0.26, 0.61), 0, 4.44, 0.01, COLORS.Silver, Enum.Material.Metal, "head", HEAD)
    B("Head_Faceplate", Vector3.new(0.62, 0.43, 0.20), 0, 3.78, -0.43, COLORS.Silver, Enum.Material.Metal, "head", HEAD)
    local eyeL = B("Eye_Left", Vector3.new(0.23, 0.10, 0.08), -0.22, 4.03, -0.405, COLORS.BlueLight, Enum.Material.Neon, "head", HEAD)
    local eyeR = B("Eye_Right", Vector3.new(0.23, 0.10, 0.08), 0.22, 4.03, -0.405, COLORS.BlueLight, Enum.Material.Neon, "head", HEAD)
    table.insert(mechaEyeParts, eyeL)
    table.insert(mechaEyeParts, eyeR)
    B("Head_Cheek_Left", Vector3.new(0.17, 0.45, 0.25), -0.44, 3.88, -0.22, COLORS.Red, Enum.Material.Metal, "head", HEAD)
    B("Head_Cheek_Right", Vector3.new(0.17, 0.45, 0.25), 0.44, 3.88, -0.22, COLORS.Red, Enum.Material.Metal, "head", HEAD)
    B("Head_Antenna_Left", Vector3.new(0.13, 0.56, 0.13), -0.39, 4.51, 0.04, COLORS.Gold, Enum.Material.Metal, "head", HEAD, 0, 0, -9)
    B("Head_Antenna_Right", Vector3.new(0.13, 0.56, 0.13), 0.39, 4.51, 0.04, COLORS.Gold, Enum.Material.Metal, "head", HEAD, 0, 0, 9)
    B("Head_Back_Panel", Vector3.new(0.60, 0.48, 0.25), 0, 4.00, 0.41, COLORS.Dark, Enum.Material.Metal, "head", HEAD)

    -- Ombros e braços articulados.
    B("Shoulder_Joint_L", Vector3.new(0.74, 0.74, 0.74), -1.47, 2.29, 0, COLORS.Silver, Enum.Material.Metal, "leftArm", LSH, 0, 0, 0, Enum.PartType.Ball)
    B("Shoulder_Joint_R", Vector3.new(0.74, 0.74, 0.74), 1.47, 2.29, 0, COLORS.Silver, Enum.Material.Metal, "rightArm", RSH, 0, 0, 0, Enum.PartType.Ball)
    B("Shoulder_Armor_L", Vector3.new(1.18, 1.06, 1.12), -1.76, 2.55, 0.03, COLORS.Red, Enum.Material.Metal, "leftArm", LSH, 0, 0, -6)
    B("Shoulder_Armor_R", Vector3.new(1.18, 1.06, 1.12), 1.76, 2.55, 0.03, COLORS.Red, Enum.Material.Metal, "rightArm", RSH, 0, 0, 6)
    B("Shoulder_Cap_L", Vector3.new(0.91, 0.22, 0.86), -1.79, 3.11, -0.02, COLORS.Silver, Enum.Material.DiamondPlate, "leftArm", LSH)
    B("Shoulder_Cap_R", Vector3.new(0.91, 0.22, 0.86), 1.79, 3.11, -0.02, COLORS.Silver, Enum.Material.DiamondPlate, "rightArm", RSH)
    B("Shoulder_Stripe_L", Vector3.new(0.17, 0.74, 0.12), -2.14, 2.55, -0.55, COLORS.Gold, Enum.Material.Neon, "leftArm", LSH)
    B("Shoulder_Stripe_R", Vector3.new(0.17, 0.74, 0.12), 2.14, 2.55, -0.55, COLORS.Gold, Enum.Material.Neon, "rightArm", RSH)
    B("UpperArm_L", Vector3.new(0.80, 1.13, 0.83), -1.89, 1.67, 0, COLORS.Blue, Enum.Material.Metal, "leftArm", LSH, 0, 0, -3)
    B("UpperArm_R", Vector3.new(0.80, 1.13, 0.83), 1.89, 1.67, 0, COLORS.Blue, Enum.Material.Metal, "rightArm", RSH, 0, 0, 3)
    B("UpperArm_OuterPlate_L", Vector3.new(0.22, 0.83, 0.69), -2.30, 1.68, -0.03, COLORS.Silver, Enum.Material.Metal, "leftArm", LSH)
    B("UpperArm_OuterPlate_R", Vector3.new(0.22, 0.83, 0.69), 2.30, 1.68, -0.03, COLORS.Silver, Enum.Material.Metal, "rightArm", RSH)
    B("Elbow_L", Vector3.new(0.62, 0.38, 0.66), -1.93, 1.00, 0, COLORS.Dark, Enum.Material.Metal, "leftArm", LSH, 0, 0, 0, Enum.PartType.Ball)
    B("Elbow_R", Vector3.new(0.62, 0.38, 0.66), 1.93, 1.00, 0, COLORS.Dark, Enum.Material.Metal, "rightArm", RSH, 0, 0, 0, Enum.PartType.Ball)
    B("Forearm_L", Vector3.new(0.91, 1.05, 0.94), -1.98, 0.50, -0.02, COLORS.Red, Enum.Material.Metal, "leftArm", LSH, 0, 0, -2)
    B("Forearm_R", Vector3.new(0.91, 1.05, 0.94), 1.98, 0.50, -0.02, COLORS.Red, Enum.Material.Metal, "rightArm", RSH, 0, 0, 2)
    B("Forearm_Guard_L", Vector3.new(0.96, 0.20, 0.88), -1.98, 0.81, -0.50, COLORS.Silver, Enum.Material.DiamondPlate, "leftArm", LSH)
    B("Forearm_Guard_R", Vector3.new(0.96, 0.20, 0.88), 1.98, 0.81, -0.50, COLORS.Silver, Enum.Material.DiamondPlate, "rightArm", RSH)
    B("Wrist_L", Vector3.new(0.62, 0.30, 0.62), -1.98, -0.07, -0.03, COLORS.Silver, Enum.Material.Metal, "leftArm", LSH)
    B("Wrist_R", Vector3.new(0.62, 0.30, 0.62), 1.98, -0.07, -0.03, COLORS.Silver, Enum.Material.Metal, "rightArm", RSH)
    local handL = B("Hand_L", Vector3.new(0.72, 0.55, 0.74), -1.98, -0.43, -0.08, COLORS.Dark, Enum.Material.Metal, "leftArm", LSH)
    local handR = B("Hand_R", Vector3.new(0.72, 0.55, 0.74), 1.98, -0.43, -0.08, COLORS.Dark, Enum.Material.Metal, "rightArm", RSH)
    for _, fingerSpec in ipairs({
        {"L", -1.98, "leftArm", LSH, -1},
        {"R", 1.98, "rightArm", RSH, 1},
    }) do
        local side, x, groupName, pivot, sign = fingerSpec[1], fingerSpec[2], fingerSpec[3], fingerSpec[4], fingerSpec[5]
        for finger = 1, 3 do
            local fx = x + (finger - 2) * 0.20
            B("Finger_" .. side .. "_" .. finger, Vector3.new(0.14, 0.29, 0.18), fx, -0.79, -0.12, COLORS.Silver, Enum.Material.Metal, groupName, pivot, 0, 0, 0)
        end
        B("Thumb_" .. side, Vector3.new(0.17, 0.30, 0.20), x + sign * 0.40, -0.45, -0.16, COLORS.Silver, Enum.Material.Metal, groupName, pivot, 0, 0, sign * 22)
    end
    table.insert(mechaEmitters, addSparkEmitter(handL, ColorSequence.new(COLORS.BlueLight, COLORS.Gold), 3))
    table.insert(mechaEmitters, addSparkEmitter(handR, ColorSequence.new(COLORS.BlueLight, COLORS.Gold), 3))
    table.insert(mechaTrails, createTrail(handL, COLORS.BlueLight))
    table.insert(mechaTrails, createTrail(handR, COLORS.BlueLight))
    local shoulderL = B("Shoulder_FX_Node_L", Vector3.new(0.18, 0.18, 0.18), -2.17, 2.50, -0.45, COLORS.BlueLight, Enum.Material.Neon, "leftArm", LSH, 0, 0, 0, Enum.PartType.Ball, 0.15)
    local shoulderR = B("Shoulder_FX_Node_R", Vector3.new(0.18, 0.18, 0.18), 2.17, 2.50, -0.45, COLORS.BlueLight, Enum.Material.Neon, "rightArm", RSH, 0, 0, 0, Enum.PartType.Ball, 0.15)
    table.insert(mechaEmitters, addSparkEmitter(shoulderL, ColorSequence.new(COLORS.Gold, COLORS.White), CFG.FX_PARTICLE_RATE))
    table.insert(mechaEmitters, addSparkEmitter(shoulderR, ColorSequence.new(COLORS.Gold, COLORS.White), CFG.FX_PARTICLE_RATE))

    -- Pernas: coxa, joelho, canela, pistões e pés grandes.
    B("Hip_Joint_L", Vector3.new(0.78, 0.78, 0.82), -0.65, -0.68, 0, COLORS.Silver, Enum.Material.Metal, "leftLeg", LHIP, 0, 0, 0, Enum.PartType.Ball)
    B("Hip_Joint_R", Vector3.new(0.78, 0.78, 0.82), 0.65, -0.68, 0, COLORS.Silver, Enum.Material.Metal, "rightLeg", RHIP, 0, 0, 0, Enum.PartType.Ball)
    B("Thigh_L", Vector3.new(0.98, 1.10, 1.00), -0.70, -1.34, 0.03, COLORS.Red, Enum.Material.Metal, "leftLeg", LHIP, 0, 0, -2)
    B("Thigh_R", Vector3.new(0.98, 1.10, 1.00), 0.70, -1.34, 0.03, COLORS.Red, Enum.Material.Metal, "rightLeg", RHIP, 0, 0, 2)
    B("Thigh_FrontPlate_L", Vector3.new(0.66, 0.70, 0.16), -0.70, -1.31, -0.54, COLORS.Silver, Enum.Material.DiamondPlate, "leftLeg", LHIP)
    B("Thigh_FrontPlate_R", Vector3.new(0.66, 0.70, 0.16), 0.70, -1.31, -0.54, COLORS.Silver, Enum.Material.DiamondPlate, "rightLeg", RHIP)
    B("Knee_L", Vector3.new(0.76, 0.52, 0.78), -0.70, -1.94, -0.04, COLORS.Dark, Enum.Material.Metal, "leftLeg", LHIP, 0, 0, 0, Enum.PartType.Ball)
    B("Knee_R", Vector3.new(0.76, 0.52, 0.78), 0.70, -1.94, -0.04, COLORS.Dark, Enum.Material.Metal, "rightLeg", RHIP, 0, 0, 0, Enum.PartType.Ball)
    B("Knee_Cap_L", Vector3.new(0.52, 0.36, 0.18), -0.70, -1.94, -0.48, COLORS.Gold, Enum.Material.Metal, "leftLeg", LHIP)
    B("Knee_Cap_R", Vector3.new(0.52, 0.36, 0.18), 0.70, -1.94, -0.48, COLORS.Gold, Enum.Material.Metal, "rightLeg", RHIP)
    B("Shin_L", Vector3.new(1.00, 1.22, 1.06), -0.73, -2.56, 0.05, COLORS.Blue, Enum.Material.Metal, "leftLeg", LHIP, 0, 0, -2)
    B("Shin_R", Vector3.new(1.00, 1.22, 1.06), 0.73, -2.56, 0.05, COLORS.Blue, Enum.Material.Metal, "rightLeg", RHIP, 0, 0, 2)
    B("Shin_Armor_L", Vector3.new(0.74, 0.74, 0.15), -0.73, -2.51, -0.55, COLORS.Silver, Enum.Material.DiamondPlate, "leftLeg", LHIP)
    B("Shin_Armor_R", Vector3.new(0.74, 0.74, 0.15), 0.73, -2.51, -0.55, COLORS.Silver, Enum.Material.DiamondPlate, "rightLeg", RHIP)
    B("Shin_Side_L", Vector3.new(0.20, 0.82, 0.65), -1.25, -2.51, 0.05, COLORS.Red, Enum.Material.Metal, "leftLeg", LHIP)
    B("Shin_Side_R", Vector3.new(0.20, 0.82, 0.65), 1.25, -2.51, 0.05, COLORS.Red, Enum.Material.Metal, "rightLeg", RHIP)
    B("Ankle_L", Vector3.new(0.67, 0.34, 0.70), -0.73, -3.20, 0.04, COLORS.Steel, Enum.Material.Metal, "leftLeg", LHIP)
    B("Ankle_R", Vector3.new(0.67, 0.34, 0.70), 0.73, -3.20, 0.04, COLORS.Steel, Enum.Material.Metal, "rightLeg", RHIP)
    B("Foot_L", Vector3.new(1.20, 0.48, 1.48), -0.78, -3.39, -0.34, COLORS.Dark, Enum.Material.Metal, "leftLeg", LHIP)
    B("Foot_R", Vector3.new(1.20, 0.48, 1.48), 0.78, -3.39, -0.34, COLORS.Dark, Enum.Material.Metal, "rightLeg", RHIP)
    B("Foot_Toe_Plate_L", Vector3.new(0.94, 0.22, 0.55), -0.78, -3.31, -0.85, COLORS.Red, Enum.Material.Metal, "leftLeg", LHIP)
    B("Foot_Toe_Plate_R", Vector3.new(0.94, 0.22, 0.55), 0.78, -3.31, -0.85, COLORS.Red, Enum.Material.Metal, "rightLeg", RHIP)
    B("Foot_Light_L", Vector3.new(0.22, 0.10, 0.08), -0.78, -3.31, -1.14, COLORS.BlueLight, Enum.Material.Neon, "leftLeg", LHIP)
    B("Foot_Light_R", Vector3.new(0.22, 0.10, 0.08), 0.78, -3.31, -1.14, COLORS.BlueLight, Enum.Material.Neon, "rightLeg", RHIP)
    B("Piston_L", Vector3.new(0.20, 0.95, 0.20), -0.34, -2.51, 0.58, COLORS.Silver, Enum.Material.Metal, "leftLeg", LHIP, 0, 0, -8, Enum.PartType.Cylinder)
    B("Piston_R", Vector3.new(0.20, 0.95, 0.20), 0.34, -2.51, 0.58, COLORS.Silver, Enum.Material.Metal, "rightLeg", RHIP, 0, 0, 8, Enum.PartType.Cylinder)

    -- Pneus nas laterais dos ombros para reforçar a leitura veículo/robô.
    B("Shoulder_Wheel_L", Vector3.new(0.95, 0.48, 0.95), -2.24, 2.23, 0.14, COLORS.Tire, Enum.Material.SmoothPlastic, "leftArm", LSH, 0, 0, 90, Enum.PartType.Cylinder)
    B("Shoulder_Wheel_R", Vector3.new(0.95, 0.48, 0.95), 2.24, 2.23, 0.14, COLORS.Tire, Enum.Material.SmoothPlastic, "rightArm", RSH, 0, 0, 90, Enum.PartType.Cylinder)
    B("Shoulder_Wheel_Hub_L", Vector3.new(0.43, 0.12, 0.43), -2.51, 2.23, 0.14, COLORS.Silver, Enum.Material.Metal, "leftArm", LSH, 0, 0, 90, Enum.PartType.Cylinder)
    B("Shoulder_Wheel_Hub_R", Vector3.new(0.43, 0.12, 0.43), 2.51, 2.23, 0.14, COLORS.Silver, Enum.Material.Metal, "rightArm", RSH, 0, 0, 90, Enum.PartType.Cylinder)

    -- Linhas de painel, parafusos e entradas de ar: muitos detalhes pequenos, mas baratos.
    for i = 1, 4 do
        local y = 2.98 - i * 0.12
        B("Chest_Vent_L_" .. i, Vector3.new(0.28, 0.045, 0.04), -1.04, y, -0.82, COLORS.Dark, Enum.Material.Metal)
        B("Chest_Vent_R_" .. i, Vector3.new(0.28, 0.045, 0.04), 1.04, y, -0.82, COLORS.Dark, Enum.Material.Metal)
    end
    for i = 1, 3 do
        B("Belt_Detail_" .. i, Vector3.new(0.13, 0.16, 0.05), (i - 2) * 0.24, 0.12, -0.55, COLORS.Gold, Enum.Material.Neon)
    end

    -- Fumaça de pouso reutilizável nos pés.
    local footL = nil
    local footR = nil
    for _, item in ipairs(parts) do
        if item.name == "Foot_L" then footL = item.part end
        if item.name == "Foot_R" then footR = item.part end
    end
    if footL then table.insert(mechaSmokeEmitters, addSmokeEmitter(footL)) end
    if footR then table.insert(mechaSmokeEmitters, addSmokeEmitter(footR)) end
end

---------------------------
-- 4. CONSTRUÇÃO PROCEDURAL DO VEÍCULO
---------------------------
local vehicleModel = createVisualModel("__TransformerEngine_Vehicle_" .. player.UserId)
local vehicleParts = {}
local vehicleEmitters = {}
local vehicleLights = {}
local vehicleWheelPivots = {}

local function buildVehicle()
    local m = vehicleModel
    local parts = vehicleParts

    local function B(name, size, x, y, z, color, material, group, pivot, rx, ry, rz, shape, transparency, className)
        local cf = CFrame.new(x, y, z) * angleCF(rx, ry, rz)
        local p = newPart(m, parts, name, size, cf, color, material, shape, group, pivot, transparency, className)
        return p
    end

    B("Vehicle_Underframe", Vector3.new(2.85, 0.45, 5.60), 0, -0.55, 0.05, COLORS.Black, Enum.Material.Metal)
    B("Vehicle_Chassis", Vector3.new(3.20, 0.62, 6.05), 0, -0.30, 0, COLORS.Dark, Enum.Material.Metal)
    B("Vehicle_Cab_Main", Vector3.new(3.15, 1.25, 3.05), 0, 0.46, -0.73, COLORS.Red, Enum.Material.Metal)
    B("Vehicle_Hood", Vector3.new(3.08, 0.45, 1.92), 0, 0.51, -2.48, COLORS.Blue, Enum.Material.Metal)
    B("Vehicle_Hood_Center", Vector3.new(0.72, 0.13, 1.55), 0, 0.77, -2.45, COLORS.Silver, Enum.Material.DiamondPlate)
    B("Vehicle_Hood_Stripe_L", Vector3.new(0.16, 0.07, 1.45), -0.95, 0.76, -2.47, COLORS.Gold, Enum.Material.Neon)
    B("Vehicle_Hood_Stripe_R", Vector3.new(0.16, 0.07, 1.45), 0.95, 0.76, -2.47, COLORS.Gold, Enum.Material.Neon)
    B("Vehicle_Roof", Vector3.new(2.75, 0.34, 1.85), 0, 1.31, -0.54, COLORS.Blue, Enum.Material.Metal)
    B("Vehicle_Windshield", Vector3.new(2.55, 0.77, 0.10), 0, 0.92, -1.66, COLORS.Glass, Enum.Material.Glass, "body", Vector3.zero, -22, 0, 0, nil, 0.14)
    B("Vehicle_Rear_Window", Vector3.new(2.45, 0.55, 0.10), 0, 0.94, 0.36, COLORS.Glass, Enum.Material.Glass, "body", Vector3.zero, 14, 0, 0, nil, 0.20)
    B("Vehicle_Side_Window_L", Vector3.new(0.09, 0.56, 1.12), -1.59, 0.96, -0.62, COLORS.Glass, Enum.Material.Glass, "body", Vector3.zero, 0, 0, -5, nil, 0.16)
    B("Vehicle_Side_Window_R", Vector3.new(0.09, 0.56, 1.12), 1.59, 0.96, -0.62, COLORS.Glass, Enum.Material.Glass, "body", Vector3.zero, 0, 0, 5, nil, 0.16)

    -- Para-brisa com bordas e carroceria lateral.
    B("Vehicle_A_Pillar_L", Vector3.new(0.15, 0.91, 0.17), -1.35, 0.91, -1.52, COLORS.Silver, Enum.Material.Metal, "body", Vector3.zero, -20, 0, -12)
    B("Vehicle_A_Pillar_R", Vector3.new(0.15, 0.91, 0.17), 1.35, 0.91, -1.52, COLORS.Silver, Enum.Material.Metal, "body", Vector3.zero, -20, 0, 12)
    B("Vehicle_Door_L", Vector3.new(0.18, 0.68, 1.55), -1.61, 0.28, -0.15, COLORS.Blue, Enum.Material.Metal)
    B("Vehicle_Door_R", Vector3.new(0.18, 0.68, 1.55), 1.61, 0.28, -0.15, COLORS.Blue, Enum.Material.Metal)
    B("Vehicle_Door_Handle_L", Vector3.new(0.07, 0.07, 0.30), -1.73, 0.36, -0.20, COLORS.Silver, Enum.Material.Metal)
    B("Vehicle_Door_Handle_R", Vector3.new(0.07, 0.07, 0.30), 1.73, 0.36, -0.20, COLORS.Silver, Enum.Material.Metal)
    B("Vehicle_Front_Bumper", Vector3.new(3.25, 0.30, 0.32), 0, -0.18, -3.40, COLORS.Silver, Enum.Material.Metal)
    B("Vehicle_Front_Grille", Vector3.new(1.12, 0.46, 0.10), 0, 0.05, -3.34, COLORS.Black, Enum.Material.Metal)
    B("Vehicle_Grille_Bar_1", Vector3.new(0.93, 0.055, 0.04), 0, 0.17, -3.405, COLORS.Silver, Enum.Material.Metal)
    B("Vehicle_Grille_Bar_2", Vector3.new(0.93, 0.055, 0.04), 0, 0.04, -3.405, COLORS.Silver, Enum.Material.Metal)
    B("Vehicle_Grille_Bar_3", Vector3.new(0.93, 0.055, 0.04), 0, -0.09, -3.405, COLORS.Silver, Enum.Material.Metal)
    local headL = B("Vehicle_Headlight_L", Vector3.new(0.52, 0.24, 0.12), -1.07, 0.15, -3.35, COLORS.BlueLight, Enum.Material.Neon)
    local headR = B("Vehicle_Headlight_R", Vector3.new(0.52, 0.24, 0.12), 1.07, 0.15, -3.35, COLORS.BlueLight, Enum.Material.Neon)
    table.insert(vehicleLights, buildPointLight(headL, COLORS.BlueLight, 9, 1.3))
    table.insert(vehicleLights, buildPointLight(headR, COLORS.BlueLight, 9, 1.3))
    B("Vehicle_Tail_L", Vector3.new(0.47, 0.25, 0.10), -1.22, 0.12, 3.06, COLORS.RedLight, Enum.Material.Neon)
    B("Vehicle_Tail_R", Vector3.new(0.47, 0.25, 0.10), 1.22, 0.12, 3.06, COLORS.RedLight, Enum.Material.Neon)
    B("Vehicle_Rear_Bumper", Vector3.new(3.14, 0.25, 0.30), 0, -0.20, 3.17, COLORS.Steel, Enum.Material.Metal)
    B("Vehicle_Side_Skirt_L", Vector3.new(0.18, 0.20, 2.55), -1.62, -0.33, 0.08, COLORS.Silver, Enum.Material.Metal)
    B("Vehicle_Side_Skirt_R", Vector3.new(0.18, 0.20, 2.55), 1.62, -0.33, 0.08, COLORS.Silver, Enum.Material.Metal)
    B("Vehicle_Roof_Lightbar", Vector3.new(1.28, 0.10, 0.18), 0, 1.54, -0.45, COLORS.BlueLight, Enum.Material.Neon)
    B("Vehicle_Mirror_L", Vector3.new(0.30, 0.20, 0.24), -1.79, 0.80, -1.13, COLORS.Black, Enum.Material.Metal)
    B("Vehicle_Mirror_R", Vector3.new(0.30, 0.20, 0.24), 1.79, 0.80, -1.13, COLORS.Black, Enum.Material.Metal)
    B("Vehicle_Exhaust_L", Vector3.new(0.22, 0.62, 0.22), -1.00, -0.48, 2.75, COLORS.Silver, Enum.Material.Metal, "body", Vector3.zero, 90, 0, 0, Enum.PartType.Cylinder)
    B("Vehicle_Exhaust_R", Vector3.new(0.22, 0.62, 0.22), 1.00, -0.48, 2.75, COLORS.Silver, Enum.Material.Metal, "body", Vector3.zero, 90, 0, 0, Enum.PartType.Cylinder)
    local exhaustL = B("Vehicle_Exhaust_Glow_L", Vector3.new(0.12, 0.12, 0.12), -1.00, -0.78, 2.75, COLORS.BlueLight, Enum.Material.Neon, "body", Vector3.zero, 0, 0, 0, Enum.PartType.Ball)
    local exhaustR = B("Vehicle_Exhaust_Glow_R", Vector3.new(0.12, 0.12, 0.12), 1.00, -0.78, 2.75, COLORS.BlueLight, Enum.Material.Neon, "body", Vector3.zero, 0, 0, 0, Enum.PartType.Ball)
    table.insert(vehicleEmitters, addSparkEmitter(exhaustL, ColorSequence.new(COLORS.BlueLight, COLORS.White), 2))
    table.insert(vehicleEmitters, addSparkEmitter(exhaustR, ColorSequence.new(COLORS.BlueLight, COLORS.White), 2))

    -- Quatro rodas geradas em código, com cubos centrais e parafusos.
    local wheelSpecs = {
        {"wheelFL", -1.48, -0.82, -2.02, -1},
        {"wheelFR", 1.48, -0.82, -2.02, 1},
        {"wheelRL", -1.48, -0.82, 2.02, -1},
        {"wheelRR", 1.48, -0.82, 2.02, 1},
    }
    for _, spec in ipairs(wheelSpecs) do
        local group, x, y, z, side = spec[1], spec[2], spec[3], spec[4], spec[5]
        local pivot = Vector3.new(x, y, z)
        vehicleWheelPivots[group] = pivot
        B("Tire_" .. group, Vector3.new(1.30, 0.60, 1.30), x, y, z, COLORS.Tire, Enum.Material.SmoothPlastic, group, pivot, 0, 0, 90, Enum.PartType.Cylinder)
        B("Hub_" .. group, Vector3.new(0.72, 0.14, 0.72), x + side * 0.34, y, z, COLORS.Silver, Enum.Material.Metal, group, pivot, 0, 0, 90, Enum.PartType.Cylinder)
        B("Hub_Core_" .. group, Vector3.new(0.28, 0.16, 0.28), x + side * 0.43, y, z, COLORS.BlueLight, Enum.Material.Neon, group, pivot, 0, 0, 90, Enum.PartType.Cylinder)
        for bolt = 1, 6 do
            local a = (bolt / 6) * math.pi * 2
            local by = y + math.cos(a) * 0.24
            local bz = z + math.sin(a) * 0.24
            B("WheelBolt_" .. group .. "_" .. bolt, Vector3.new(0.09, 0.09, 0.09), x + side * 0.45, by, bz, COLORS.Gold, Enum.Material.Metal, group, pivot, 0, 0, 0, Enum.PartType.Ball)
        end
    end

    -- Iluminação inferior discreta.
    local underGlowL = B("Vehicle_UnderGlow_L", Vector3.new(0.38, 0.05, 2.10), -1.18, -0.70, 0.10, COLORS.BlueLight, Enum.Material.Neon, "body", Vector3.zero, 0, 0, 0, nil, 0.35)
    local underGlowR = B("Vehicle_UnderGlow_R", Vector3.new(0.38, 0.05, 2.10), 1.18, -0.70, 0.10, COLORS.BlueLight, Enum.Material.Neon, "body", Vector3.zero, 0, 0, 0, nil, 0.35)
    table.insert(vehicleLights, buildPointLight(underGlowL, COLORS.BlueLight, 7, 0.6))
    table.insert(vehicleLights, buildPointLight(underGlowR, COLORS.BlueLight, 7, 0.6))
end

safeCall(buildMecha)
safeCall(buildVehicle)

---------------------------
-- 5. EFEITOS DE TELA E INTERFACE MOBILE
---------------------------
local oldGui = playerGui:FindFirstChild("TransformerEngineUI")
if oldGui then oldGui:Destroy() end

local screenGui = Instance.new("ScreenGui")
screenGui.Name = "TransformerEngineUI"
screenGui.ResetOnSpawn = false
screenGui.IgnoreGuiInset = false
screenGui.DisplayOrder = 35
screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
screenGui.Parent = playerGui

local blur = Lighting:FindFirstChild("TransformerEngine_Blur")
if blur then blur:Destroy() end
blur = Instance.new("BlurEffect")
blur.Name = "TransformerEngine_Blur"
blur.Size = 0
blur.Parent = Lighting

local hud = Instance.new("Frame")
hud.Name = "StatusPanel"
hud.AnchorPoint = Vector2.new(0.5, 0)
hud.Position = UDim2.new(0.5, 0, 0, 10)
hud.Size = UDim2.new(0, 285, 0, 62)
hud.BackgroundColor3 = Color3.fromRGB(8, 14, 24)
hud.BackgroundTransparency = 0.18
hud.BorderSizePixel = 0
hud.Parent = screenGui
local hudCorner = Instance.new("UICorner")
hudCorner.CornerRadius = UDim.new(0, 10)
hudCorner.Parent = hud
local hudStroke = Instance.new("UIStroke")
hudStroke.Color = COLORS.BlueLight
hudStroke.Thickness = 1.2
hudStroke.Transparency = 0.18
hudStroke.Parent = hud

local title = Instance.new("TextLabel")
title.Name = "EngineTitle"
title.BackgroundTransparency = 1
title.Position = UDim2.new(0, 12, 0, 5)
title.Size = UDim2.new(1, -24, 0, 22)
title.Font = Enum.Font.GothamBlack
title.Text = "TRANSFORMER ENGINE"
title.TextColor3 = COLORS.White
title.TextSize = 15
title.TextXAlignment = Enum.TextXAlignment.Left
title.Parent = hud

local status = Instance.new("TextLabel")
status.Name = "StatusText"
status.BackgroundTransparency = 1
status.Position = UDim2.new(0, 12, 0, 27)
status.Size = UDim2.new(1, -24, 0, 18)
status.Font = Enum.Font.GothamMedium
status.Text = "SISTEMA ONLINE  •  HUMANOIDE"
status.TextColor3 = COLORS.BlueLight
status.TextSize = 10
status.TextXAlignment = Enum.TextXAlignment.Left
status.Parent = hud

local barBack = Instance.new("Frame")
barBack.Name = "ProgressBack"
barBack.Position = UDim2.new(0, 12, 1, -10)
barBack.Size = UDim2.new(1, -24, 0, 3)
barBack.BackgroundColor3 = Color3.fromRGB(38, 49, 64)
barBack.BorderSizePixel = 0
barBack.Visible = false
barBack.Parent = hud
local barFill = Instance.new("Frame")
barFill.Name = "ProgressFill"
barFill.Size = UDim2.new(0, 0, 1, 0)
barFill.BackgroundColor3 = COLORS.BlueLight
barFill.BorderSizePixel = 0
barFill.Parent = barBack

local buttonInfo = {
    {form = FORMS.MECHA, text = "ROBÔ", color = COLORS.Blue, key = Enum.KeyCode.T},
    {form = FORMS.VEICULO, text = "VEÍCULO", color = COLORS.Red, key = Enum.KeyCode.V},
    {form = FORMS.HUMANOIDE, text = "HUMANO", color = COLORS.Steel, key = Enum.KeyCode.H},
}
local buttons = {}

local function makeButton(info, index)
    local button = Instance.new("TextButton")
    button.Name = "TransformTo_" .. info.form
    button.AnchorPoint = Vector2.new(1, 0)
    button.Position = UDim2.new(1, -12, 0.39, (index - 1) * 55)
    button.Size = UDim2.new(0, 104, 0, 46)
    button.BackgroundColor3 = Color3.fromRGB(9, 17, 29)
    button.BackgroundTransparency = 0.10
    button.BorderSizePixel = 0
    button.AutoButtonColor = true
    button.Font = Enum.Font.GothamBlack
    button.Text = info.text
    button.TextColor3 = COLORS.White
    button.TextSize = 12
    button.Selectable = false
    button.Parent = screenGui

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 9)
    corner.Parent = button

    local stroke = Instance.new("UIStroke")
    stroke.Name = "ButtonStroke"
    stroke.Color = info.color
    stroke.Thickness = 1.4
    stroke.Transparency = 0.15
    stroke.Parent = button

    local line = Instance.new("Frame")
    line.Name = "Accent"
    line.BorderSizePixel = 0
    line.BackgroundColor3 = info.color
    line.Size = UDim2.new(0, 4, 0.70, 0)
    line.Position = UDim2.new(0, 6, 0.15, 0)
    line.Parent = button
    local lineCorner = Instance.new("UICorner")
    lineCorner.CornerRadius = UDim.new(1, 0)
    lineCorner.Parent = line

    button.Activated:Connect(function()
        if not destroyed then
            -- Botões Activated funcionam por toque e por mouse.
            if _G.__TransformerEngineRequest then
                _G.__TransformerEngineRequest(info.form)
            end
        end
    end)

    buttons[info.form] = {button = button, stroke = stroke, info = info}
end

for i, info in ipairs(buttonInfo) do
    makeButton(info, i)
end

local hint = Instance.new("TextLabel")
hint.Name = "TouchHint"
hint.AnchorPoint = Vector2.new(1, 0)
hint.Position = UDim2.new(1, -12, 0.39, 166)
hint.Size = UDim2.new(0, 150, 0, 22)
hint.BackgroundTransparency = 1
hint.Font = Enum.Font.Gotham
hint.Text = "TOQUE PARA TRANSFORMAR"
hint.TextColor3 = Color3.fromRGB(172, 192, 211)
hint.TextSize = 8
hint.TextXAlignment = Enum.TextXAlignment.Right
hint.Parent = screenGui

---------------------------
-- 6. AVATAR ORIGINAL: CAPTURA E VISIBILIDADE LOCAL
---------------------------
local function capturePart(part)
    if part:IsA("BasePart") and avatarParts[part] == nil then
        avatarParts[part] = part.LocalTransparencyModifier
    end
end

local function captureCharacter(char)
    if descendantConnection then
        descendantConnection:Disconnect()
        descendantConnection = nil
    end

    character = char
    humanoid = char:WaitForChild("Humanoid", 12)
    hrp = char:WaitForChild("HumanoidRootPart", 12)
    avatarParts = {}

    for _, descendant in ipairs(char:GetDescendants()) do
        capturePart(descendant)
    end

    descendantConnection = char.DescendantAdded:Connect(function(descendant)
        capturePart(descendant)
        if form ~= FORMS.HUMANOIDE and descendant:IsA("BasePart") then
            descendant.LocalTransparencyModifier = 1
        end
    end)

    if humanoid then
        normalProperties = {
            WalkSpeed = humanoid.WalkSpeed,
            JumpPower = humanoid.JumpPower,
            JumpHeight = humanoid.JumpHeight,
            UseJumpPower = humanoid.UseJumpPower,
            AutoRotate = humanoid.AutoRotate,
            CameraOffset = humanoid.CameraOffset,
        }
    end
end

local function setAvatarVisible(visibleAmount)
    visibleAmount = clamp01(visibleAmount)
    for part, original in pairs(avatarParts) do
        if part and part.Parent then
            part.LocalTransparencyModifier = 1 - (1 - original) * visibleAmount
        end
    end
end

local function saveMovementIfNeeded()
    if humanoid and not normalProperties then
        normalProperties = {
            WalkSpeed = humanoid.WalkSpeed,
            JumpPower = humanoid.JumpPower,
            JumpHeight = humanoid.JumpHeight,
            UseJumpPower = humanoid.UseJumpPower,
            AutoRotate = humanoid.AutoRotate,
            CameraOffset = humanoid.CameraOffset,
        }
    end
end

local function applyFormMovement(target)
    if not humanoid then return end
    saveMovementIfNeeded()

    if target == FORMS.HUMANOIDE then
        if normalProperties then
            humanoid.WalkSpeed = normalProperties.WalkSpeed or CFG.HUMANOID_SPEED
            humanoid.UseJumpPower = normalProperties.UseJumpPower
            humanoid.JumpPower = normalProperties.JumpPower or 50
            humanoid.JumpHeight = normalProperties.JumpHeight or 7.2
            humanoid.AutoRotate = normalProperties.AutoRotate ~= false
            humanoid.CameraOffset = normalProperties.CameraOffset or Vector3.zero
        else
            humanoid.WalkSpeed = CFG.HUMANOID_SPEED
            humanoid.JumpPower = 50
            humanoid.JumpHeight = 7.2
            humanoid.AutoRotate = true
        end
    elseif target == FORMS.MECHA then
        humanoid.WalkSpeed = CFG.MECHA_SPEED
        humanoid.JumpPower = CFG.MECHA_JUMP_POWER
        humanoid.JumpHeight = CFG.MECHA_JUMP_HEIGHT
        humanoid.AutoRotate = true
        humanoid.CameraOffset = Vector3.new(0, 1.0, 0)
    elseif target == FORMS.VEICULO then
        humanoid.WalkSpeed = CFG.VEHICLE_SPEED
        humanoid.JumpPower = 0
        humanoid.JumpHeight = 0
        humanoid.AutoRotate = true
        humanoid.CameraOffset = Vector3.new(0, 0.15, 0)
    end
end

---------------------------
-- 7. ANIMAÇÃO PROCEDURAL DAS PEÇAS
---------------------------
local function groupTransform(item, mode, phase, moving, cameraLocalLook, now)
    local group = item.group
    local pivot = item.pivot or Vector3.zero
    local rx, ry, rz = 0, 0, 0

    if mode == FORMS.MECHA then
        if group == "leftArm" then
            rx = moving and math.sin(phase + math.pi) * CFG.WALK_SWAY or math.sin(now * 1.15) * 0.025
            rz = moving and -0.06 or math.sin(now * 0.8) * 0.018
        elseif group == "rightArm" then
            rx = moving and math.sin(phase) * CFG.WALK_SWAY or math.sin(now * 1.15 + 1) * 0.025
            rz = moving and 0.06 or math.sin(now * 0.8 + 1) * 0.018
        elseif group == "leftLeg" then
            rx = moving and math.sin(phase) * 0.46 or 0
        elseif group == "rightLeg" then
            rx = moving and math.sin(phase + math.pi) * 0.46 or 0
        elseif group == "head" then
            ry = math.clamp(-cameraLocalLook.X * 0.43, -0.52, 0.52)
            rx = math.clamp(cameraLocalLook.Y * 0.20, -0.18, 0.18)
        end
    elseif mode == FORMS.VEICULO then
        if string.sub(group, 1, 5) == "wheel" then
            local spin = wheelPhase
            local steer = 0
            if group == "wheelFL" or group == "wheelFR" then
                steer = moving and math.clamp(-cameraLocalLook.X * 0.12, -0.24, 0.24) or 0
            end
            local base = CFrame.new(pivot)
            return base * CFrame.Angles(spin, steer, 0) * CFrame.new(-pivot)
        end
    end

    return CFrame.new(pivot) * CFrame.Angles(rx, ry, rz) * CFrame.new(-pivot)
end

local function updateModel(parts, rootCF, visible, assemble, mode, phase, moving, cameraLocalLook, now, bob)
    visible = clamp01(visible)
    assemble = clamp01(assemble)
    if visible <= 0.002 then
        -- Continua oculto sem recalcular centenas de CFrames.
        for _, item in ipairs(parts) do
            if item.part and item.part.Parent and item.part.Transparency < 0.999 then
                item.part.Transparency = 1
            end
        end
        return
    end

    local rootMotion = CFrame.new(0, bob or 0, 0)
    for _, item in ipairs(parts) do
        local p = item.part
        if p and p.Parent then
            local spread = item.scatter:Lerp(CFrame.new(), assemble)
            local pose = groupTransform(item, mode, phase, moving, cameraLocalLook, now)
            p.CFrame = rootCF * rootMotion * spread * pose * item.baseCF
            p.Transparency = 1 - visible * (1 - item.baseTransparency)
        end
    end
end

local function setEffectsEnabled(enabled, vehicleEnabled)
    for _, emitter in ipairs(mechaEmitters) do
        if emitter and emitter.Parent then emitter.Enabled = enabled end
    end
    for _, trail in ipairs(mechaTrails) do
        if trail and trail.Parent then trail.Enabled = enabled and vehicleEnabled == false end
    end
    for _, emitter in ipairs(mechaSmokeEmitters) do
        if emitter and emitter.Parent then emitter.Enabled = false end
    end
    for _, emitter in ipairs(vehicleEmitters) do
        if emitter and emitter.Parent then emitter.Enabled = enabled and vehicleEnabled == true end
    end
end

local function emitTransformationBurst()
    for _, emitter in ipairs(mechaEmitters) do
        if emitter and emitter.Parent then safeCall(function() emitter:Emit(7) end) end
    end
    for _, emitter in ipairs(vehicleEmitters) do
        if emitter and emitter.Parent then safeCall(function() emitter:Emit(5) end) end
    end
    for _, emitter in ipairs(mechaSmokeEmitters) do
        if emitter and emitter.Parent then safeCall(function() emitter:Emit(10) end) end
    end
end

---------------------------
-- 8. CONTROLE DE ESTADO E TRANSFORMAÇÃO
---------------------------
local function updateButtonStates()
    for target, entry in pairs(buttons) do
        local selected = (not transition and form == target)
        entry.stroke.Thickness = selected and 2.6 or 1.2
        entry.stroke.Transparency = selected and 0 or 0.22
        entry.button.BackgroundColor3 = selected and Color3.fromRGB(21, 40, 60) or Color3.fromRGB(9, 17, 29)
        entry.button.TextTransparency = transition and 0.28 or 0
    end
end

local function requestForm(target)
    if destroyed or transition or not buttons[target] then return end
    if target == form then return end
    if not humanoid or not hrp or humanoid.Health <= 0 then
        status.Text = "AGUARDANDO AVATAR..."
        status.TextColor3 = COLORS.Gold
        return
    end

    saveMovementIfNeeded()
    transition = {
        from = form,
        to = target,
        startTime = os.clock(),
        duration = CFG.TRANSFORM_TIME,
        progress = 0,
    }

    -- Imobilização suave durante o processo, sem ancorar o personagem.
    humanoid.WalkSpeed = 0
    humanoid.JumpPower = 0
    humanoid.JumpHeight = 0
    humanoid:Move(Vector3.zero, true)
    if hrp then
        hrp.AssemblyLinearVelocity = Vector3.zero
        hrp.AssemblyAngularVelocity = Vector3.zero
        lastRootCF = hrp.CFrame
    end

    status.Text = "SISTEMA DE TRANSFORMAÇÃO ATIVADO..."
    status.TextColor3 = COLORS.Gold
    barBack.Visible = true
    barFill.Size = UDim2.new(0, 0, 1, 0)
    updateButtonStates()
    setEffectsEnabled(true, target == FORMS.VEICULO)
    emitTransformationBurst()
end

_G.__TransformerEngineRequest = requestForm

local function completeTransition()
    if not transition then return end
    local target = transition.to
    form = target
    transition = nil
    applyFormMovement(form)

    if form == FORMS.HUMANOIDE then
        setAvatarVisible(1)
        status.Text = "SISTEMA ONLINE  •  HUMANOIDE"
        status.TextColor3 = COLORS.BlueLight
    elseif form == FORMS.MECHA then
        setAvatarVisible(0)
        status.Text = "MODO MECHA  •  NÚCLEO ENERGON ATIVO"
        status.TextColor3 = COLORS.BlueLight
        for _, emitter in ipairs(mechaSmokeEmitters) do
            if emitter and emitter.Parent then safeCall(function() emitter:Emit(8) end) end
        end
    else
        setAvatarVisible(0)
        status.Text = "MODO VEÍCULO  •  PROPULSÃO ONLINE"
        status.TextColor3 = COLORS.RedLight
    end

    barBack.Visible = false
    barFill.Size = UDim2.new(0, 0, 1, 0)
    blur.Size = 0
    setEffectsEnabled(false, form == FORMS.VEICULO)
    updateButtonStates()
end

---------------------------
-- 9. INICIALIZAÇÃO DO AVATAR E RESPAWN
---------------------------
local function bindCharacter(char)
    local ok = pcall(function()
        captureCharacter(char)
    end)
    if not ok then return end

    if humanoid and hrp then
        if form == FORMS.HUMANOIDE then
            setAvatarVisible(1)
        else
            setAvatarVisible(0)
            applyFormMovement(form)
        end
    end
end

if player.Character then
    task.spawn(bindCharacter, player.Character)
end
player.CharacterAdded:Connect(function(char)
    task.spawn(function()
        bindCharacter(char)
    end)
end)

---------------------------
-- 10. ENTRADA: TECLADO OPCIONAL E TOQUE MOBILE
---------------------------
UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed or destroyed or not CFG.SHOW_KEYBOARD_SHORTCUTS then return end
    if input.KeyCode == Enum.KeyCode.T then
        requestForm(FORMS.MECHA)
    elseif input.KeyCode == Enum.KeyCode.V then
        requestForm(FORMS.VEICULO)
    elseif input.KeyCode == Enum.KeyCode.H then
        requestForm(FORMS.HUMANOIDE)
    end
end)

---------------------------
-- 11. LOOP VISUAL ÚNICO (OTIMIZADO)
---------------------------
local renderConnection
renderConnection = RunService.RenderStepped:Connect(function(deltaTime)
    if destroyed then return end
    camera = Workspace.CurrentCamera or camera

    if not character or not character.Parent or not humanoid or not hrp or not hrp.Parent then
        status.Text = "AGUARDANDO PERSONAGEM..."
        return
    end

    local now = os.clock()
    local moveMagnitude = humanoid.MoveDirection.Magnitude
    local moving = moveMagnitude > 0.05 and not transition
    for _, trail in ipairs(mechaTrails) do
        if trail and trail.Parent then
            trail.Enabled = (not transition) and form == FORMS.MECHA and moving and moveMagnitude > 0.45
        end
    end
    local speedScale = 1
    if form == FORMS.VEICULO then
        speedScale = 1.65
    elseif form == FORMS.MECHA then
        speedScale = 1.05
    end
    if moving then
        walkPhase += deltaTime * (form == FORMS.VEICULO and 11 or 7.3) * math.clamp(moveMagnitude, 0, 1)
        wheelPhase += deltaTime * (form == FORMS.VEICULO and 7.5 or 0)
    end
    effectTick += deltaTime

    local rootCF = hrp.CFrame
    local look = camera and camera.CFrame.LookVector or rootCF.LookVector
    local cameraLocalLook = rootCF:VectorToObjectSpace(look)

    local mVis, mAssembly = 0, 1
    local vVis, vAssembly = 0, 1
    local avatarVisible = form == FORMS.HUMANOIDE and 1 or 0
    local blurAmount = 0
    local progress = 0
    local bob = moving and math.abs(math.sin(walkPhase * 2)) * (form == FORMS.MECHA and 0.08 or 0.025) or 0

    if transition then
        progress = clamp01((now - transition.startTime) / transition.duration)
        transition.progress = progress
        local eased = smoothstep(progress)
        if transition.from == FORMS.MECHA then
            mVis = 1 - eased
            mAssembly = 1 - eased
        elseif transition.to == FORMS.MECHA then
            mVis = eased
            mAssembly = eased
        end
        if transition.from == FORMS.VEICULO then
            vVis = 1 - eased
            vAssembly = 1 - eased
        elseif transition.to == FORMS.VEICULO then
            vVis = eased
            vAssembly = eased
        end

        if transition.from == FORMS.HUMANOIDE then
            avatarVisible = 1 - eased
        elseif transition.to == FORMS.HUMANOIDE then
            avatarVisible = eased
        else
            avatarVisible = 0
        end

        blurAmount = math.sin(progress * math.pi) * 7
        barFill.Size = UDim2.new(progress, 0, 1, 0)
        status.Text = "TRANSFORMANDO  •  " .. tostring(math.floor(progress * 100 + 0.5)) .. "%"
        if progress >= 1 then
            completeTransition()
            mVis = form == FORMS.MECHA and 1 or 0
            mAssembly = 1
            vVis = form == FORMS.VEICULO and 1 or 0
            vAssembly = 1
            avatarVisible = form == FORMS.HUMANOIDE and 1 or 0
            blurAmount = 0
        end
    else
        if form == FORMS.MECHA then mVis = 1 end
        if form == FORMS.VEICULO then vVis = 1 end
        if form == FORMS.HUMANOIDE then avatarVisible = 1 end
    end

    setAvatarVisible(avatarVisible)

    local mechaRootCF = rootCF * CFrame.new(0, 0.62, 0)
    local vehicleRootCF = rootCF * CFrame.new(0, -1.45, 0)
    local mechaBob = form == FORMS.MECHA and bob or (transition and (transition.to == FORMS.MECHA or transition.from == FORMS.MECHA) and math.sin(progress * math.pi * 6) * 0.035 or 0)

    updateModel(mechaParts, mechaRootCF, mVis, mAssembly, FORMS.MECHA, walkPhase, moving and form == FORMS.MECHA, cameraLocalLook, now, mechaBob)
    updateModel(vehicleParts, vehicleRootCF, vVis, vAssembly, FORMS.VEICULO, wheelPhase, moving and form == FORMS.VEICULO, cameraLocalLook, now, 0)

    if mechaCoreLight and mechaCoreLight.Parent then
        local active = form == FORMS.MECHA or (transition and (transition.from == FORMS.MECHA or transition.to == FORMS.MECHA))
        local pulse = 2.0 + (math.sin(now * 3.2) + 1) * 0.5
        mechaCoreLight.Brightness = active and pulse or 0
        mechaCoreLight.Range = active and (11 + math.sin(now * 2.0) * 1.3) or 0
    end
    for _, eye in ipairs(mechaEyeParts) do
        if eye and eye.Parent then
            local active = form == FORMS.MECHA or (transition and transition.to == FORMS.MECHA)
            eye.Color = active and COLORS.BlueLight or COLORS.Steel
        end
    end
    for _, light in ipairs(vehicleLights) do
        if light and light.Parent then
            light.Brightness = (form == FORMS.VEICULO or (transition and transition.to == FORMS.VEICULO)) and (1.0 + math.sin(now * 4) * 0.18) or 0
        end
    end

    local desiredFOV = CFG.HUMANOID_FOV
    if form == FORMS.MECHA then desiredFOV = CFG.MECHA_FOV end
    if form == FORMS.VEICULO then desiredFOV = CFG.VEHICLE_FOV end
    if transition then
        local fromFOV = transition.from == FORMS.MECHA and CFG.MECHA_FOV or (transition.from == FORMS.VEICULO and CFG.VEHICLE_FOV or CFG.HUMANOID_FOV)
        local toFOV = transition.to == FORMS.MECHA and CFG.MECHA_FOV or (transition.to == FORMS.VEICULO and CFG.VEHICLE_FOV or CFG.HUMANOID_FOV)
        desiredFOV = fromFOV + (toFOV - fromFOV) * smoothstep(progress)
    end
    if camera then
        camera.FieldOfView = camera.FieldOfView + (desiredFOV - camera.FieldOfView) * math.clamp(deltaTime * 3.5, 0, 1)
    end
    if blur then
        blur.Size = blur.Size + (blurAmount - blur.Size) * math.clamp(deltaTime * 9, 0, 1)
    end

    if not transition then
        updateButtonStates()
    end
end)

---------------------------
-- 12. ELEGÂNCIA: LIMPEZA QUANDO O SCRIPT É REMOVIDO
---------------------------
local function cleanup()
    if destroyed then return end
    destroyed = true
    if renderConnection then renderConnection:Disconnect() end
    if descendantConnection then descendantConnection:Disconnect() end
    if humanoid and normalProperties then
        pcall(function()
            humanoid.WalkSpeed = normalProperties.WalkSpeed or CFG.HUMANOID_SPEED
            humanoid.UseJumpPower = normalProperties.UseJumpPower
            humanoid.JumpPower = normalProperties.JumpPower or 50
            humanoid.JumpHeight = normalProperties.JumpHeight or 7.2
            humanoid.AutoRotate = normalProperties.AutoRotate ~= false
            humanoid.CameraOffset = normalProperties.CameraOffset or Vector3.zero
        end)
    end
    for part, original in pairs(avatarParts) do
        if part and part.Parent then
            pcall(function() part.LocalTransparencyModifier = original end)
        end
    end
    if mechaModel and mechaModel.Parent then mechaModel:Destroy() end
    if vehicleModel and vehicleModel.Parent then vehicleModel:Destroy() end
    if screenGui and screenGui.Parent then screenGui:Destroy() end
    if blur and blur.Parent then blur:Destroy() end
    if _G.__TransformerEngineRequest == requestForm then
        _G.__TransformerEngineRequest = nil
    end
end

script.AncestryChanged:Connect(function(_, parent)
    if parent == nil then cleanup() end
end)

updateButtonStates()
print("[Transformer Engine] Autônomo iniciado. Toque em ROBÔ, VEÍCULO ou HUMANO.")
