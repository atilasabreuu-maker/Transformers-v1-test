--[[
    TRANSFORMER ENGINE MOBILE - LocalScript único
    Versão: 1.0

    INSTALAÇÃO:
    1. Coloque este LocalScript em StarterPlayer > StarterPlayerScripts.
    2. Em ReplicatedStorage, crie os Models "MechaRig" e "VehicleRig".
    3. MechaRig precisa ter uma BasePart chamada HumanoidRootPart, definida como PrimaryPart,
       peças ligadas por Motor6D e, idealmente, um Humanoid + Animator.
    4. VehicleRig precisa ter uma PrimaryPart (preferencialmente chamada VehicleRoot ou Chassis).
       O veículo é controlado cinematicamente pelo script; suas peças são ancoradas localmente.
    5. Sons opcionais: ReplicatedStorage > Sounds > Sound_Transform_Start,
       Sound_Transform_Mid, Sound_Transform_End, Sound_Footstep_Mecha, Sound_Vehicle_Engine.
       Cada Sound precisa ter SoundId configurado.
    6. Animações opcionais: ReplicatedStorage > Animations > Animation objects nomeados
       MechaIdle, MechaWalk, MechaRun e MechaJump. Os rigs precisam usar juntas compatíveis.

    LIMITAÇÃO IMPORTANTE:
    Este é um sistema visual/controlado pelo cliente. Um LocalScript não pode garantir que
    outros jogadores vejam os clones ou controlar NetworkOwnership de um modelo arbitrário.
    Replicação multiplayer real requer uma camada de servidor/RemoteEvent.
]]

--==============================================================
-- 1. SERVIÇOS E CONFIGURAÇÕES
--==============================================================
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local ContextActionService = game:GetService("ContextActionService")
local Lighting = game:GetService("Lighting")
local Workspace = game:GetService("Workspace")
local Debris = game:GetService("Debris")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")
local camera = Workspace.CurrentCamera

local CONFIG = {
    MechaTemplateName = "MechaRig",
    VehicleTemplateName = "VehicleRig",
    SoundsFolderName = "Sounds",
    AnimationsFolderName = "Animations",

    TransformDuration = 3.35,
    TransformHumanDuration = 2.65,
    MechaWalkSpeed = 27,
    MechaRunSpeed = 36,
    MechaJumpVelocity = 46,
    VehicleMaxSpeed = 66,
    VehicleReverseSpeed = 20,
    VehicleAcceleration = 44,
    VehicleBrake = 78,
    VehicleTurnSpeed = 2.2,
    VehicleIdleTurnSpeed = 0.62,
    Gravity = Workspace.Gravity,

    MechaFOV = 74,
    VehicleFOV = 79,
    HumanFOV = 70,
    CameraShake = 0.22,
    CameraShakeFrequency = 9,
    GroundRayUp = 180,
    GroundRayDown = 650,
    WallRayHeight = 2.4,

    EnableProceduralHeadTracking = true,
    EnableCameraShake = true,
    EnableParticleEffects = true,
    EnableSoundEffects = true,
    EnableKeyboardShortcuts = true,
    AutoPlayAuthoredAnimations = true,

    SparkTexture = "rbxasset://textures/particles/sparkles_main.dds",
    SmokeTexture = "rbxasset://textures/particles/smoke_main.dds",
    AccentColor = Color3.fromRGB(38, 220, 255),
    WarningColor = Color3.fromRGB(255, 126, 54),
}

local STATE_HUMAN = "Humanoide"
local STATE_TRANSFORMING = "Transformando"
local STATE_MECHA = "Mecha"
local STATE_VEHICLE = "Veículo"

local currentState = STATE_HUMAN
local isBusy = false
local activeRig = nil
local character = player.Character or player.CharacterAdded:Wait()
local humanoid = character:WaitForChild("Humanoid")
local humanRoot = character:WaitForChild("HumanoidRootPart")
local humanCache = nil
local activeLoopSound = nil
local footstepClock = 0
local jumpRequested = false
local verticalVelocity = 0
local moveSpeedCurrent = 0
local vehicleSpeedCurrent = 0
local sessionClock = 0
local destroying = false

--==============================================================
-- 2. UTILITÁRIOS
--==============================================================
local function safeCall(callback, ...)
    local ok, result = pcall(callback, ...)
    if not ok then
        warn("[Transformer Engine]", result)
    end
    return ok, result
end

local function clamp01(value)
    return math.clamp(value, 0, 1)
end

local function smoothStep(value)
    value = clamp01(value)
    return value * value * (3 - 2 * value)
end

local function easeOutCubic(value)
    value = clamp01(value)
    return 1 - ((1 - value) ^ 3)
end

local function easeOutBack(value)
    value = clamp01(value)
    local c1 = 1.70158
    local c3 = c1 + 1
    return 1 + c3 * ((value - 1) ^ 3) + c1 * ((value - 1) ^ 2)
end

local function lerpNumber(a, b, alpha)
    return a + (b - a) * alpha
end

local function getBaseParts(model)
    local parts = {}
    if not model then
        return parts
    end
    for _, item in ipairs(model:GetDescendants()) do
        if item:IsA("BasePart") then
            table.insert(parts, item)
        end
    end
    return parts
end

local function findBasePart(model, names)
    if not model then
        return nil
    end
    for _, name in ipairs(names) do
        local found = model:FindFirstChild(name, true)
        if found and found:IsA("BasePart") then
            return found
        end
    end
    return nil
end

local function getMotorJoints(model)
    local joints = {}
    for _, item in ipairs(model:GetDescendants()) do
        if item:IsA("Motor6D") and item.Part0 and item.Part1 then
            table.insert(joints, item)
        end
    end
    return joints
end

local function makeRayParams(excludes)
    local params = RaycastParams.new()
    params.FilterType = Enum.RaycastFilterType.Exclude
    params.FilterDescendantsInstances = excludes or {}
    params.IgnoreWater = false
    return params
end

local function raycastGround(position, excludes)
    local origin = position + Vector3.new(0, CONFIG.GroundRayUp, 0)
    local direction = Vector3.new(0, -(CONFIG.GroundRayUp + CONFIG.GroundRayDown), 0)
    local result = Workspace:Raycast(origin, direction, makeRayParams(excludes))
    if result then
        return result.Position.Y, result
    end
    return position.Y - 3, nil
end

-- Calcula a altura inferior do modelo em relação ao pivot, considerando a orientação das peças.
local function calculateBottomOffset(model)
    local pivot = model:GetPivot()
    local lowest = 0
    local foundPart = false
    for _, part in ipairs(getBaseParts(model)) do
        local localCF = pivot:ToObjectSpace(part.CFrame)
        local halfHeight = math.abs(localCF.RightVector.Y) * part.Size.X * 0.5
            + math.abs(localCF.UpVector.Y) * part.Size.Y * 0.5
            + math.abs(localCF.LookVector.Y) * part.Size.Z * 0.5
        local partBottom = localCF.Position.Y - halfHeight
        if not foundPart or partBottom < lowest then
            lowest = partBottom
            foundPart = true
        end
    end
    if not foundPart then
        return -3
    end
    return lowest
end

local function getYaw(cf)
    local _, yaw, _ = cf:ToOrientation()
    return yaw
end

local function flatUnit(vector)
    local flattened = Vector3.new(vector.X, 0, vector.Z)
    if flattened.Magnitude < 0.001 then
        return Vector3.zero
    end
    return flattened.Unit
end

local function getMoveVector()
    -- Usa o mesmo vetor de controle do joystick padrão do Roblox, também no touch.
    local ok, module = pcall(function()
        local scripts = player:FindFirstChild("PlayerScripts")
        local playerModule = scripts and scripts:FindFirstChild("PlayerModule")
        if not playerModule then
            return nil
        end
        return require(playerModule)
    end)
    if ok and module then
        local okControls, controls = pcall(function()
            return module:GetControls()
        end)
        if okControls and controls then
            local okVector, vector = pcall(function()
                return controls:GetMoveVector()
            end)
            if okVector and typeof(vector) == "Vector3" then
                return vector
            end
        end
    end
    return Vector3.zero
end

local function getCameraRelativeDirection(moveVector)
    if not camera then
        return Vector3.zero
    end
    local cameraForward = flatUnit(camera.CFrame.LookVector)
    local cameraRight = flatUnit(camera.CFrame.RightVector)
    local result = cameraRight * moveVector.X + cameraForward * (-moveVector.Z)
    if result.Magnitude > 1 then
        result = result.Unit
    end
    return result
end

local function tweenCameraFOV(target, duration)
    camera = Workspace.CurrentCamera or camera
    if not camera then
        return
    end
    safeCall(function()
        TweenService:Create(
            camera,
            TweenInfo.new(duration or 0.45, Enum.EasingStyle.Quad, Enum.EasingDirection.Out),
            { FieldOfView = target }
        ):Play()
    end)
end

--==============================================================
-- 3. SOM, LUZES E EFEITOS
--==============================================================
local function getSoundTemplate(soundName)
    local folder = ReplicatedStorage:FindFirstChild(CONFIG.SoundsFolderName)
    if not folder then
        folder = ReplicatedStorage:FindFirstChild("MechaSounds")
    end
    if folder then
        local sound = folder:FindFirstChild(soundName)
        if sound and sound:IsA("Sound") then
            return sound
        end
    end
    return nil
end

local function stopLoopSound()
    if activeLoopSound then
        safeCall(function()
            activeLoopSound:Stop()
            activeLoopSound:Destroy()
        end)
        activeLoopSound = nil
    end
end

local function playSound(soundName, parentPart, looped)
    if not CONFIG.EnableSoundEffects or not parentPart then
        return nil
    end
    local template = getSoundTemplate(soundName)
    if not template then
        return nil
    end
    local sound = template:Clone()
    sound.Name = "Local_" .. soundName
    sound.Looped = looped == true
    sound.RollOffMaxDistance = math.max(sound.RollOffMaxDistance, 100)
    sound.Parent = parentPart
    local ok = pcall(function()
        sound:Play()
    end)
    if not ok then
        sound:Destroy()
        return nil
    end
    if looped then
        activeLoopSound = sound
    else
        Debris:AddItem(sound, math.max(sound.TimeLength + 2, 5))
    end
    return sound
end

local function getOrCreateAttachment(part, name, position)
    local attachment = part:FindFirstChild(name)
    if attachment and attachment:IsA("Attachment") then
        return attachment
    end
    attachment = Instance.new("Attachment")
    attachment.Name = name
    attachment.Position = position or Vector3.zero
    attachment.Parent = part
    return attachment
end

local function addEmitter(attachment, name, texture, color, rate)
    local emitter = Instance.new("ParticleEmitter")
    emitter.Name = name
    emitter.Texture = texture
    emitter.Color = ColorSequence.new(color)
    emitter.LightEmission = 0.8
    emitter.Brightness = 2
    emitter.Lifetime = NumberRange.new(0.22, 0.65)
    emitter.Speed = NumberRange.new(3, 10)
    emitter.SpreadAngle = Vector2.new(160, 160)
    emitter.Rotation = NumberRange.new(0, 360)
    emitter.RotSpeed = NumberRange.new(-160, 160)
    emitter.Size = NumberSequence.new({
        NumberSequenceKeypoint.new(0, 0.22),
        NumberSequenceKeypoint.new(0.35, 0.12),
        NumberSequenceKeypoint.new(1, 0),
    })
    emitter.Transparency = NumberSequence.new({
        NumberSequenceKeypoint.new(0, 0.05),
        NumberSequenceKeypoint.new(0.65, 0.2),
        NumberSequenceKeypoint.new(1, 1),
    })
    emitter.Rate = rate or 0
    emitter.Enabled = false
    emitter.Parent = attachment
    return emitter
end

local function emitTransformEffects(rigData, amount)
    if not CONFIG.EnableParticleEffects or not rigData or not rigData.effects then
        return
    end
    for _, emitter in ipairs(rigData.effects.emitters or {}) do
        if emitter and emitter.Parent then
            safeCall(function()
                emitter:Emit(amount or 14)
            end)
        end
    end
end

local function prepareMechaEffects(rigData)
    if not rigData or not rigData.model then
        return
    end
    local model = rigData.model
    local chest = findBasePart(model, { "UpperTorso", "Torso", "Chest", "Body", "HumanoidRootPart" })
    local leftShoulder = findBasePart(model, { "LeftUpperArm", "LeftArm", "LeftShoulder", "HumanoidRootPart" })
    local rightShoulder = findBasePart(model, { "RightUpperArm", "RightArm", "RightShoulder", "HumanoidRootPart" })
    local leftFoot = findBasePart(model, { "LeftFoot", "LeftLowerLeg", "LeftLeg", "HumanoidRootPart" })
    local rightFoot = findBasePart(model, { "RightFoot", "RightLowerLeg", "RightLeg", "HumanoidRootPart" })
    local head = findBasePart(model, { "Head" })
    local effects = { emitters = {}, light = nil, selection = nil, trails = {} }

    if chest then
        local reactorAttachment = getOrCreateAttachment(chest, "TE_ReactorAttachment", Vector3.new(0, 0, -chest.Size.Z * 0.52))
        local light = Instance.new("PointLight")
        light.Name = "TE_ReactorLight"
        light.Color = CONFIG.AccentColor
        light.Brightness = 1.6
        light.Range = 22
        light.Shadows = false
        light.Parent = reactorAttachment
        effects.light = light
        table.insert(effects.emitters, addEmitter(reactorAttachment, "TE_ReactorSparks", CONFIG.SparkTexture, CONFIG.AccentColor, 0))
    end

    if leftShoulder then
        local attachment = getOrCreateAttachment(leftShoulder, "TE_LeftShoulderFX", Vector3.new(0, leftShoulder.Size.Y * 0.45, 0))
        table.insert(effects.emitters, addEmitter(attachment, "TE_LeftSparks", CONFIG.SparkTexture, CONFIG.WarningColor, 0))
    end
    if rightShoulder then
        local attachment = getOrCreateAttachment(rightShoulder, "TE_RightShoulderFX", Vector3.new(0, rightShoulder.Size.Y * 0.45, 0))
        table.insert(effects.emitters, addEmitter(attachment, "TE_RightSparks", CONFIG.SparkTexture, CONFIG.AccentColor, 0))
    end
    if leftFoot then
        local attachment = getOrCreateAttachment(leftFoot, "TE_LeftFootSmoke", Vector3.new(0, -leftFoot.Size.Y * 0.48, 0))
        table.insert(effects.emitters, addEmitter(attachment, "TE_LeftSmoke", CONFIG.SmokeTexture, Color3.fromRGB(160, 175, 190), 0))
    end
    if rightFoot then
        local attachment = getOrCreateAttachment(rightFoot, "TE_RightFootSmoke", Vector3.new(0, -rightFoot.Size.Y * 0.48, 0))
        table.insert(effects.emitters, addEmitter(attachment, "TE_RightSmoke", CONFIG.SmokeTexture, Color3.fromRGB(160, 175, 190), 0))
    end

    -- Trilhas dos pulsos: só são criadas se existirem attachments com orientação adequada.
    for _, armName in ipairs({ "LeftLowerArm", "RightLowerArm", "LeftHand", "RightHand" }) do
        local arm = findBasePart(model, { armName })
        if arm then
            local a0 = getOrCreateAttachment(arm, "TE_TrailA", Vector3.new(0, arm.Size.Y * 0.4, 0))
            local a1 = getOrCreateAttachment(arm, "TE_TrailB", Vector3.new(0, -arm.Size.Y * 0.4, 0))
            local trail = Instance.new("Trail")
            trail.Name = "TE_WristTrail_" .. armName
            trail.Attachment0 = a0
            trail.Attachment1 = a1
            trail.Color = ColorSequence.new(CONFIG.AccentColor, CONFIG.WarningColor)
            trail.Lifetime = 0.16
            trail.LightEmission = 0.8
            trail.Enabled = false
            trail.Parent = arm
            table.insert(effects.trails, trail)
        end
    end

    if head then
        local box = Instance.new("SelectionBox")
        box.Name = "TE_TransformationOutline"
        box.Adornee = head
        box.Color3 = CONFIG.AccentColor
        box.LineThickness = 0.035
        box.SurfaceTransparency = 0.94
        box.Visible = false
        box.Parent = model
        effects.selection = box
    end
    rigData.effects = effects
end

--==============================================================
-- 4. INTERFACE MOBILE
--==============================================================
local oldGui = playerGui:FindFirstChild("TransformerEngineUI")
if oldGui then
    oldGui:Destroy()
end

local screenGui = Instance.new("ScreenGui")
screenGui.Name = "TransformerEngineUI"
screenGui.ResetOnSpawn = false
screenGui.IgnoreGuiInset = true
screenGui.DisplayOrder = 40
screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
screenGui.Parent = playerGui

local mainPanel = Instance.new("Frame")
mainPanel.Name = "StatusPanel"
mainPanel.AnchorPoint = Vector2.new(0, 0)
mainPanel.Position = UDim2.new(0, 16, 0, 16)
mainPanel.Size = UDim2.new(0, 245, 0, 74)
mainPanel.BackgroundColor3 = Color3.fromRGB(9, 16, 25)
mainPanel.BackgroundTransparency = 0.13
mainPanel.BorderSizePixel = 0
mainPanel.Parent = screenGui
local panelCorner = Instance.new("UICorner")
panelCorner.CornerRadius = UDim.new(0, 10)
panelCorner.Parent = mainPanel
local panelStroke = Instance.new("UIStroke")
panelStroke.Color = CONFIG.AccentColor
panelStroke.Transparency = 0.22
panelStroke.Thickness = 1.2
panelStroke.Parent = mainPanel

local panelTitle = Instance.new("TextLabel")
panelTitle.Name = "Title"
panelTitle.BackgroundTransparency = 1
panelTitle.Position = UDim2.new(0, 12, 0, 7)
panelTitle.Size = UDim2.new(1, -24, 0, 19)
panelTitle.Font = Enum.Font.GothamBold
panelTitle.Text = "TRANSFORMER ENGINE"
panelTitle.TextColor3 = CONFIG.AccentColor
panelTitle.TextSize = 13
panelTitle.TextXAlignment = Enum.TextXAlignment.Left
panelTitle.Parent = mainPanel

local statusLabel = Instance.new("TextLabel")
statusLabel.Name = "Status"
statusLabel.BackgroundTransparency = 1
statusLabel.Position = UDim2.new(0, 12, 0, 30)
statusLabel.Size = UDim2.new(1, -24, 0, 18)
statusLabel.Font = Enum.Font.GothamMedium
statusLabel.Text = "SISTEMA ONLINE // HUMANO"
statusLabel.TextColor3 = Color3.fromRGB(222, 235, 244)
statusLabel.TextSize = 10
statusLabel.TextXAlignment = Enum.TextXAlignment.Left
statusLabel.Parent = mainPanel

local statusDetail = Instance.new("TextLabel")
statusDetail.Name = "Detail"
statusDetail.BackgroundTransparency = 1
statusDetail.Position = UDim2.new(0, 12, 0, 49)
statusDetail.Size = UDim2.new(1, -24, 0, 15)
statusDetail.Font = Enum.Font.Code
statusDetail.Text = "MOBILITY SYSTEM // READY"
statusDetail.TextColor3 = Color3.fromRGB(128, 161, 180)
statusDetail.TextSize = 9
statusDetail.TextXAlignment = Enum.TextXAlignment.Left
statusDetail.Parent = mainPanel

local progressBack = Instance.new("Frame")
progressBack.Name = "ProgressBack"
progressBack.AnchorPoint = Vector2.new(0, 0)
progressBack.Position = UDim2.new(0, 16, 0, 99)
progressBack.Size = UDim2.new(0, 245, 0, 5)
progressBack.BackgroundColor3 = Color3.fromRGB(29, 42, 53)
progressBack.BorderSizePixel = 0
progressBack.Visible = false
progressBack.Parent = screenGui
local progressBackCorner = Instance.new("UICorner")
progressBackCorner.CornerRadius = UDim.new(1, 0)
progressBackCorner.Parent = progressBack

local progressFill = Instance.new("Frame")
progressFill.Name = "ProgressFill"
progressFill.Size = UDim2.fromScale(0, 1)
progressFill.BackgroundColor3 = CONFIG.AccentColor
progressFill.BorderSizePixel = 0
progressFill.Parent = progressBack
local progressFillCorner = Instance.new("UICorner")
progressFillCorner.CornerRadius = UDim.new(1, 0)
progressFillCorner.Parent = progressFill

local function createActionButton(name, text, position, size, color)
    local button = Instance.new("TextButton")
    button.Name = name
    button.AnchorPoint = Vector2.new(1, 1)
    button.Position = position
    button.Size = size
    button.BackgroundColor3 = Color3.fromRGB(12, 24, 34)
    button.BackgroundTransparency = 0.06
    button.BorderSizePixel = 0
    button.AutoButtonColor = true
    button.Font = Enum.Font.GothamBold
    button.Text = text
    button.TextColor3 = color or CONFIG.AccentColor
    button.TextSize = 12
    button.TextWrapped = true
    button.Parent = screenGui
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 12)
    corner.Parent = button
    local stroke = Instance.new("UIStroke")
    stroke.Color = color or CONFIG.AccentColor
    stroke.Transparency = 0.16
    stroke.Thickness = 1.35
    stroke.Parent = button
    return button
end

-- Os botões ficam acima da área típica do botão de pulo do Roblox.
local transformButton = createActionButton(
    "TransformButton", "TRANSFORMAR", UDim2.new(1, -18, 1, -244), UDim2.new(0, 154, 0, 54), CONFIG.AccentColor
)
local humanButton = createActionButton(
    "HumanButton", "VOLTAR AO HUMANO", UDim2.new(1, -18, 1, -180), UDim2.new(0, 154, 0, 43), Color3.fromRGB(255, 186, 97)
)
local jumpButton = createActionButton(
    "JumpButton", "SALTO / BOOST", UDim2.new(1, -18, 1, -124), UDim2.new(0, 126, 0, 42), Color3.fromRGB(225, 240, 250)
)

local blur = Lighting:FindFirstChild("TransformerEngineBlur")
if not blur or not blur:IsA("BlurEffect") then
    blur = Instance.new("BlurEffect")
    blur.Name = "TransformerEngineBlur"
    blur.Size = 0
    blur.Parent = Lighting
end

local function setStatus(mainText, detailText, color)
    statusLabel.Text = mainText or ""
    statusDetail.Text = detailText or ""
    if color then
        statusLabel.TextColor3 = color
    end
end

local function updateButtons()
    if isBusy then
        transformButton.Text = "TRANSFORMAÇÃO..."
        transformButton.Active = false
        transformButton.AutoButtonColor = false
        humanButton.Active = false
        humanButton.AutoButtonColor = false
        jumpButton.Active = false
        jumpButton.AutoButtonColor = false
    else
        transformButton.Active = true
        transformButton.AutoButtonColor = true
        humanButton.Active = true
        humanButton.AutoButtonColor = true
        jumpButton.Active = currentState == STATE_MECHA or currentState == STATE_VEHICLE
        jumpButton.AutoButtonColor = jumpButton.Active
        if currentState == STATE_HUMAN then
            transformButton.Text = "INICIAR TRANSFORMAÇÃO"
        elseif currentState == STATE_MECHA then
            transformButton.Text = "TRANSFORMAR EM VEÍCULO"
        else
            transformButton.Text = "TRANSFORMAR EM ROBÔ"
        end
    end
end

--==============================================================
-- 5. CAPTURA E RESTAURAÇÃO DO AVATAR HUMANO
--==============================================================
local function captureHumanState()
    character = player.Character or character
    humanoid = character and character:FindFirstChildOfClass("Humanoid")
    humanRoot = character and character:FindFirstChild("HumanoidRootPart")
    if not character or not humanoid or not humanRoot then
        return nil
    end

    local cache = {
        character = character,
        humanoid = humanoid,
        root = humanRoot,
        rootCF = humanRoot.CFrame,
        rootBottomOffset = 0,
        parts = {},
        motors = {},
        humanoidProps = {
            WalkSpeed = humanoid.WalkSpeed,
            JumpPower = humanoid.JumpPower,
            JumpHeight = humanoid.JumpHeight,
            UseJumpPower = humanoid.UseJumpPower,
            AutoRotate = humanoid.AutoRotate,
            PlatformStand = humanoid.PlatformStand,
            Sit = humanoid.Sit,
        },
        suspended = false,
    }

    for _, item in ipairs(character:GetDescendants()) do
        if item:IsA("Motor6D") then
            table.insert(cache.motors, {
                motor = item,
                enabled = item.Enabled,
                transform = item.Transform,
            })
        end
    end

    local lowest = 0
    local hasBodyPart = false
    for _, part in ipairs(getBaseParts(character)) do
        local relative = humanRoot.CFrame:ToObjectSpace(part.CFrame)
        local halfHeight = math.abs(relative.RightVector.Y) * part.Size.X * 0.5
            + math.abs(relative.UpVector.Y) * part.Size.Y * 0.5
            + math.abs(relative.LookVector.Y) * part.Size.Z * 0.5
        local partBottom = relative.Position.Y - halfHeight
        -- Acessórios não devem definir a altura da origem humana.
        local isCoreBody = part.Name == "HumanoidRootPart" or part.Name == "Head"
            or part.Name == "UpperTorso" or part.Name == "LowerTorso"
            or string.find(part.Name, "Arm") or string.find(part.Name, "Leg")
            or string.find(part.Name, "Hand") or string.find(part.Name, "Foot")
        if isCoreBody then
            if not hasBodyPart or partBottom < lowest then
                lowest = partBottom
                hasBodyPart = true
            end
        end
        table.insert(cache.parts, {
            part = part,
            anchored = part.Anchored,
            canCollide = part.CanCollide,
            canTouch = part.CanTouch,
            canQuery = part.CanQuery,
            massless = part.Massless,
            transparency = part.Transparency,
            localTransparencyModifier = part.LocalTransparencyModifier,
            relativeCF = relative,
            startCF = part.CFrame,
        })
    end
    cache.rootBottomOffset = hasBodyPart and lowest or -2.8
    return cache
end

local function suspendHuman()
    if humanCache and humanCache.suspended then
        return true
    end
    humanCache = captureHumanState()
    if not humanCache then
        setStatus("AVATAR NÃO ENCONTRADO", "AGUARDANDO PERSONAGEM R15", CONFIG.WarningColor)
        return false
    end
    humanoid = humanCache.humanoid
    humanRoot = humanCache.root
    for _, jointData in ipairs(humanCache.motors) do
        if jointData.motor and jointData.motor.Parent then
            jointData.motor.Enabled = false
        end
    end
    for _, partData in ipairs(humanCache.parts) do
        local part = partData.part
        if part and part.Parent then
            part.Anchored = true
            part.CanCollide = false
            part.CanTouch = false
            part.LocalTransparencyModifier = 0
        end
    end
    humanoid.WalkSpeed = 0
    humanoid.JumpPower = 0
    humanoid.JumpHeight = 0
    humanoid.AutoRotate = false
    humanoid.Sit = false
    humanoid.PlatformStand = true
    humanCache.suspended = true
    return true
end

local function getHumanFoldCFrame(partName)
    local low = string.lower(partName)
    if low == "humanoidrootpart" then
        return CFrame.new(0, 0, 0)
    elseif string.find(low, "head") then
        return CFrame.new(0, 0.35, 0) * CFrame.Angles(0.1, 0, 0)
    elseif string.find(low, "torso") then
        return CFrame.new(0, 0, 0)
    elseif string.find(low, "arm") or string.find(low, "hand") then
        local sign = string.find(low, "left") and -1 or 1
        return CFrame.new(sign * 0.22, 0.08, -0.12) * CFrame.Angles(0.2, 0, sign * 0.7)
    elseif string.find(low, "leg") or string.find(low, "foot") then
        local sign = string.find(low, "left") and -1 or 1
        return CFrame.new(sign * 0.13, -0.22, 0.12) * CFrame.Angles(-0.25, 0, sign * 0.12)
    end
    return CFrame.new(0, 0, 0)
end

local function animateHumanDisassembly(alpha, destinationPivot)
    if not humanCache then
        return
    end
    local ease = easeOutCubic(alpha)
    for _, partData in ipairs(humanCache.parts) do
        local part = partData.part
        if part and part.Parent then
            local compactCF = destinationPivot * getHumanFoldCFrame(part.Name)
            part.CFrame = partData.startCF:Lerp(compactCF, ease)
            part.LocalTransparencyModifier = ease
        end
    end
end

local function placeHumanFolded(destinationPivot)
    if not humanCache then
        return
    end
    for _, partData in ipairs(humanCache.parts) do
        local part = partData.part
        if part and part.Parent then
            part.CFrame = destinationPivot * getHumanFoldCFrame(part.Name)
            part.LocalTransparencyModifier = 1
            part.Anchored = true
            part.CanCollide = false
            part.CanTouch = false
        end
    end
end

local function animateHumanAssembly(alpha, destinationPivot)
    if not humanCache then
        return
    end
    local ease = easeOutBack(alpha)
    ease = math.clamp(ease, 0, 1.08)
    for _, partData in ipairs(humanCache.parts) do
        local part = partData.part
        if part and part.Parent then
            local foldedCF = destinationPivot * getHumanFoldCFrame(part.Name)
            local finalCF = destinationPivot * partData.relativeCF
            part.CFrame = foldedCF:Lerp(finalCF, ease)
            part.LocalTransparencyModifier = 1 - clamp01(alpha)
        end
    end
end

local function restoreHuman(destinationPivot)
    if not humanCache then
        return false
    end
    local cache = humanCache
    character = cache.character
    humanoid = cache.humanoid
    humanRoot = cache.root
    if not character or not character.Parent or not humanoid or not humanRoot or not humanRoot.Parent then
        return false
    end

    -- Reposiciona as partes antes de reativar as juntas para evitar que fiquem espalhadas.
    humanRoot.CFrame = destinationPivot
    for _, partData in ipairs(cache.parts) do
        local part = partData.part
        if part and part.Parent then
            part.CFrame = destinationPivot * partData.relativeCF
        end
    end
    for _, jointData in ipairs(cache.motors) do
        local motor = jointData.motor
        if motor and motor.Parent then
            motor.Transform = jointData.transform
            motor.Enabled = jointData.enabled
        end
    end
    for _, partData in ipairs(cache.parts) do
        local part = partData.part
        if part and part.Parent then
            part.Anchored = partData.anchored
            part.CanCollide = partData.canCollide
            part.CanTouch = partData.canTouch
            part.CanQuery = partData.canQuery
            part.Massless = partData.massless
            part.LocalTransparencyModifier = partData.localTransparencyModifier
        end
    end

    local props = cache.humanoidProps
    humanoid.WalkSpeed = props.WalkSpeed
    humanoid.JumpPower = props.JumpPower
    humanoid.JumpHeight = props.JumpHeight
    humanoid.UseJumpPower = props.UseJumpPower
    humanoid.AutoRotate = props.AutoRotate
    humanoid.PlatformStand = props.PlatformStand
    humanoid.Sit = props.Sit
    safeCall(function()
        humanoid:ChangeState(Enum.HumanoidStateType.GettingUp)
    end)
    cache.suspended = false
    camera = Workspace.CurrentCamera or camera
    if camera then
        camera.CameraSubject = humanoid
    end
    tweenCameraFOV(CONFIG.HumanFOV, 0.55)
    return true
end

--==============================================================
-- 6. PREPARAÇÃO DOS MODELOS MECHA E VEÍCULO
--==============================================================
local function resolveTemplate(templateName)
    local template = ReplicatedStorage:FindFirstChild(templateName)
    if template and template:IsA("Model") then
        return template
    end
    return nil
end

local function prepareMechaModel(spawnPivot, exploded)
    local template = resolveTemplate(CONFIG.MechaTemplateName)
    if not template then
        return nil, "ReplicatedStorage.MechaRig não foi encontrado"
    end
    local model = template:Clone()
    model.Name = "Local_Transformer_Mecha"
    local root = model:FindFirstChild("HumanoidRootPart", true)
    if not root or not root:IsA("BasePart") then
        model:Destroy()
        return nil, "MechaRig precisa ter uma BasePart HumanoidRootPart"
    end
    model.PrimaryPart = root
    model.Parent = Workspace
    model:PivotTo(spawnPivot)

    local parts = getBaseParts(model)
    -- A altura é medida na pose-base, antes de aplicar offsets de montagem.
    local bottomOffset = calculateBottomOffset(model)
    local localParts = {}
    local transparency = {}
    for _, part in ipairs(parts) do
        localParts[part] = spawnPivot:ToObjectSpace(part.CFrame)
        transparency[part] = part.Transparency
        part.CanCollide = false
        part.CanTouch = false
        part.Massless = true
        part.LocalTransparencyModifier = exploded and 1 or 0
        part.Anchored = part == root
    end

    local humanoidInRig = model:FindFirstChildOfClass("Humanoid")
    if humanoidInRig then
        humanoidInRig.AutoRotate = false
        humanoidInRig.WalkSpeed = 0
        humanoidInRig.JumpPower = 0
        humanoidInRig.PlatformStand = true
        humanoidInRig.BreakJointsOnDeath = false
    end

    local motors = getMotorJoints(model)
    local motorData = {}
    for index, motor in ipairs(motors) do
        local baseTransform = motor.Transform
        local offset = CFrame.new(
            math.sin(index * 2.17) * 2.8,
            math.cos(index * 1.31) * 2.4,
            math.sin(index * 0.77) * 2.6
        ) * CFrame.Angles(math.rad((index % 3 - 1) * 45), math.rad((index % 4 - 2) * 40), math.rad((index % 5 - 2) * 28))
        local startTransform = exploded and (baseTransform * offset) or baseTransform
        table.insert(motorData, {
            motor = motor,
            base = baseTransform,
            start = startTransform,
            enabled = motor.Enabled,
        })
        motor.Transform = startTransform
        motor.Enabled = true
    end

    local data = {
        kind = STATE_MECHA,
        model = model,
        root = root,
        humanoid = humanoidInRig,
        bottomOffset = bottomOffset,
        localParts = localParts,
        transparency = transparency,
        motors = motorData,
        pivot = spawnPivot,
        effects = nil,
        tracks = {},
        activeTrackName = nil,
        transitionSequenceActive = false,
        baseYaw = getYaw(spawnPivot),
    }
    prepareMechaEffects(data)
    return data
end

local function prepareVehicleModel(spawnPivot, exploded)
    local template = resolveTemplate(CONFIG.VehicleTemplateName)
    if not template then
        return nil, "ReplicatedStorage.VehicleRig não foi encontrado"
    end
    local model = template:Clone()
    model.Name = "Local_Transformer_Vehicle"
    local primary = model.PrimaryPart
        or findBasePart(model, { "VehicleRoot", "Chassis", "Root", "HumanoidRootPart" })
    if not primary then
        model:Destroy()
        return nil, "VehicleRig precisa ter PrimaryPart ou uma peça VehicleRoot/Chassis"
    end
    model.PrimaryPart = primary
    model.Parent = Workspace
    model:PivotTo(spawnPivot)

    local localParts = {}
    local transparency = {}
    local foldedParts = {}
    local wheels = {}
    for index, part in ipairs(getBaseParts(model)) do
        local localCF = spawnPivot:ToObjectSpace(part.CFrame)
        localParts[part] = localCF
        transparency[part] = part.Transparency
        local foldedPosition = localCF.Position * 0.22
        foldedParts[part] = CFrame.new(foldedPosition) * localCF.Rotation
        part.Anchored = true
        part.CanCollide = false
        part.CanTouch = false
        part.Massless = true
        part.LocalTransparencyModifier = exploded and 1 or 0
        if string.find(string.lower(part.Name), "wheel") or string.find(string.lower(part.Name), "tire") then
            table.insert(wheels, { part = part, localCF = localCF, index = index })
        end
    end

    local data = {
        kind = STATE_VEHICLE,
        model = model,
        root = primary,
        humanoid = nil,
        bottomOffset = calculateBottomOffset(model),
        localParts = localParts,
        transparency = transparency,
        foldedParts = foldedParts,
        wheels = wheels,
        pivot = spawnPivot,
        effects = { emitters = {}, light = nil, selection = nil, trails = {} },
        baseYaw = getYaw(spawnPivot),
    }
    local glowPart = findBasePart(model, { "Headlights", "Headlight", "FrontLights", "VehicleRoot", "Chassis" }) or primary
    local attach = getOrCreateAttachment(glowPart, "TE_VehicleFX", Vector3.new(0, 0, -glowPart.Size.Z * 0.5))
    local light = Instance.new("PointLight")
    light.Name = "TE_VehicleGlow"
    light.Color = CONFIG.AccentColor
    light.Brightness = 0.85
    light.Range = 14
    light.Shadows = false
    light.Parent = attach
    data.effects.light = light
    table.insert(data.effects.emitters, addEmitter(attach, "TE_VehicleSparks", CONFIG.SparkTexture, CONFIG.AccentColor, 0))
    return data
end

local function setRigVisibility(rigData, alpha)
    if not rigData then
        return
    end
    for _, part in ipairs(getBaseParts(rigData.model)) do
        part.LocalTransparencyModifier = alpha
    end
end

local function setExplodedPose(rigData)
    if not rigData then
        return
    end
    if rigData.kind == STATE_MECHA then
        for _, motorData in ipairs(rigData.motors) do
            if motorData.motor and motorData.motor.Parent then
                motorData.motor.Transform = motorData.start
            end
        end
        setRigVisibility(rigData, 1)
    elseif rigData.kind == STATE_VEHICLE then
        local index = 0
        for part, localCF in pairs(rigData.localParts) do
            if part and part.Parent then
                index += 1
                local offset = CFrame.new(
                    math.sin(index * 1.7) * 3,
                    math.cos(index * 2.3) * 2.4,
                    math.sin(index * 1.2) * 3.2
                ) * CFrame.Angles(math.rad(index * 13 % 90), math.rad(index * 23 % 120), math.rad(index * 17 % 70))
                part.CFrame = rigData.pivot * offset * localCF
                part.LocalTransparencyModifier = 1
            end
        end
    end
end

local function animateTargetBuild(rigData, alpha)
    if not rigData then
        return
    end
    if rigData.kind == STATE_MECHA then
        local ease = easeOutBack(alpha)
        ease = math.clamp(ease, 0, 1.08)
        if not rigData.transitionSequenceActive then
            for _, motorData in ipairs(rigData.motors) do
                local motor = motorData.motor
                if motor and motor.Parent then
                    motor.Transform = motorData.start:Lerp(motorData.base, ease)
                end
            end
        end
        local visibleAlpha = clamp01(alpha)
        for part in pairs(rigData.localParts) do
            if part and part.Parent then
                part.LocalTransparencyModifier = 1 - visibleAlpha
            end
        end
        if rigData.effects and rigData.effects.selection then
            rigData.effects.selection.Visible = alpha < 0.95
        end
    elseif rigData.kind == STATE_VEHICLE then
        local ease = easeOutBack(alpha)
        ease = math.clamp(ease, 0, 1.08)
        for part, localCF in pairs(rigData.localParts) do
            if part and part.Parent then
                local foldedCF = rigData.foldedParts[part] or localCF
                part.CFrame = rigData.pivot * foldedCF:Lerp(localCF, ease)
                part.LocalTransparencyModifier = 1 - clamp01(alpha)
            end
        end
    end
end

local function animateSourceCollapse(rigData, alpha)
    if not rigData then
        return
    end
    local ease = easeOutCubic(alpha)
    if rigData.kind == STATE_MECHA then
        if not rigData.transitionSequenceActive then
            for _, motorData in ipairs(rigData.motors) do
                local motor = motorData.motor
                if motor and motor.Parent then
                    local inward = CFrame.new(0, -0.5, 0.45) * CFrame.Angles(math.rad(16), 0, math.rad(12))
                    motor.Transform = motorData.base:Lerp(motorData.base * inward, ease)
                end
            end
        end
        for part in pairs(rigData.localParts) do
            if part and part.Parent then
                part.LocalTransparencyModifier = ease
            end
        end
    elseif rigData.kind == STATE_VEHICLE then
        for part, localCF in pairs(rigData.localParts) do
            if part and part.Parent then
                local foldedCF = rigData.foldedParts[part] or (CFrame.new(localCF.Position * 0.22) * localCF.Rotation)
                part.CFrame = rigData.pivot * localCF:Lerp(foldedCF, ease)
                part.LocalTransparencyModifier = ease
            end
        end
    end
    if rigData.effects then
        if rigData.effects.selection then
            rigData.effects.selection.Visible = alpha < 0.4
        end
        if rigData.effects.light then
            rigData.effects.light.Brightness = 1.5 * (1 - ease)
        end
    end
end

--==============================================================
-- 7. ANIMAÇÕES PRÉ-GRAVADAS OPCIONAIS
--==============================================================
local function loadAuthoredAnimations(rigData)
    if not CONFIG.AutoPlayAuthoredAnimations or not rigData or rigData.kind ~= STATE_MECHA then
        return
    end
    local folder = ReplicatedStorage:FindFirstChild(CONFIG.AnimationsFolderName)
    if not folder then
        return
    end
    local animator = rigData.humanoid and rigData.humanoid:FindFirstChildOfClass("Animator")
    if not animator and rigData.humanoid then
        animator = Instance.new("Animator")
        animator.Parent = rigData.humanoid
    end
    if not animator then
        return
    end
    for _, name in ipairs({ "MechaIdle", "MechaWalk", "MechaRun", "MechaJump", "TransformationSequence" }) do
        local animation = folder:FindFirstChild(name)
        if animation and animation:IsA("Animation") and animation.AnimationId ~= "" then
            local ok, track = pcall(function()
                return animator:LoadAnimation(animation)
            end)
            if ok and track then
                track.Looped = name ~= "MechaJump" and name ~= "TransformationSequence"
                if name == "MechaIdle" then
                    track.Priority = Enum.AnimationPriority.Idle
                elseif name == "TransformationSequence" then
                    track.Priority = Enum.AnimationPriority.Action
                else
                    track.Priority = Enum.AnimationPriority.Movement
                end
                rigData.tracks[name] = track
            end
        end
    end
end

local function playTransformationTrack(rigData)
    if not rigData or not rigData.tracks then
        return false
    end
    local track = rigData.tracks.TransformationSequence
    if not track then
        return false
    end
    for name, otherTrack in pairs(rigData.tracks) do
        if name ~= "TransformationSequence" and otherTrack.IsPlaying then
            safeCall(function() otherTrack:Stop(0.12) end)
        end
    end
    rigData.transitionSequenceActive = true
    safeCall(function()
        track.TimePosition = 0
        track:Play(0.12, 1, 1)
    end)
    return true
end

local function changeMechaTrack(rigData, desiredName)
    if not rigData or not rigData.tracks then
        return false
    end
    local desired = rigData.tracks[desiredName]
    if not desired then
        -- Se o rig não possuir uma animação para esse estado, interrompe a faixa anterior
        -- para que a animação procedural possa assumir o controle das juntas.
        for _, track in pairs(rigData.tracks) do
            if track.IsPlaying then
                safeCall(function() track:Stop(0.18) end)
            end
        end
        rigData.activeTrackName = nil
        return false
    end
    if rigData.activeTrackName == desiredName and desired.IsPlaying then
        return true
    end
    for name, track in pairs(rigData.tracks) do
        if name ~= desiredName and track.IsPlaying then
            safeCall(function()
                track:Stop(0.22)
            end)
        end
    end
    safeCall(function()
        desired:Play(0.22, 1, desiredName == "MechaRun" and 1.12 or 1)
    end)
    rigData.activeTrackName = desiredName
    return true
end

local function stopRigAnimations(rigData)
    if rigData and rigData.tracks then
        for _, track in pairs(rigData.tracks) do
            safeCall(function()
                track:Stop(0.15)
            end)
        end
    end
end

--==============================================================
-- 8. POSE PROCEDURAL DO MECHA
--==============================================================
local function findMotorForPart(rigData, targetNames, motorNames)
    if not rigData or not rigData.motors then
        return nil
    end
    for _, data in ipairs(rigData.motors) do
        local motor = data.motor
        if motor and motor.Parent then
            local part1Name = motor.Part1 and motor.Part1.Name or ""
            local part0Name = motor.Part0 and motor.Part0.Name or ""
            local lowerMotor = string.lower(motor.Name)
            for _, target in ipairs(targetNames or {}) do
                if string.lower(part1Name) == string.lower(target) or string.lower(part0Name) == string.lower(target) then
                    return data
                end
            end
            for _, target in ipairs(motorNames or {}) do
                if string.find(lowerMotor, string.lower(target)) then
                    return data
                end
            end
        end
    end
    return nil
end

local function applyProceduralPose(rigData, now, moveAmount, isRunning)
    if not rigData or rigData.kind ~= STATE_MECHA then
        return
    end
    local hasAuthored = rigData.activeTrackName and rigData.tracks[rigData.activeTrackName]
        and rigData.tracks[rigData.activeTrackName].IsPlaying
    if hasAuthored then
        return
    end

    local frequency = isRunning and 8.5 or 5.7
    local amplitude = math.clamp(moveAmount, 0, 1)
    local gait = math.sin(now * frequency) * amplitude
    local opposite = math.sin(now * frequency + math.pi) * amplitude

    for _, motorData in ipairs(rigData.motors) do
        local motor = motorData.motor
        if motor and motor.Parent then
            local partName = string.lower((motor.Part1 and motor.Part1.Name) or "")
            local motorName = string.lower(motor.Name)
            local transform = motorData.base

            if string.find(partName, "leftupperleg") or string.find(partName, "leftleg")
                or string.find(motorName, "lefthip") then
                transform = transform * CFrame.Angles(gait * 0.48, 0, 0)
            elseif string.find(partName, "rightupperleg") or string.find(partName, "rightleg")
                or string.find(motorName, "righthip") then
                transform = transform * CFrame.Angles(opposite * 0.48, 0, 0)
            elseif string.find(partName, "leftupperarm") or string.find(partName, "leftarm")
                or string.find(motorName, "leftshoulder") then
                transform = transform * CFrame.Angles(opposite * 0.25, 0, math.rad(5) + gait * 0.12)
            elseif string.find(partName, "rightupperarm") or string.find(partName, "rightarm")
                or string.find(motorName, "rightshoulder") then
                transform = transform * CFrame.Angles(gait * 0.25, 0, -math.rad(5) + opposite * 0.12)
            elseif string.find(partName, "uppertorso") or string.find(partName, "torso")
                or string.find(motorName, "waist") then
                transform = transform * CFrame.Angles(math.rad(2) * amplitude, 0, math.rad(2.5) * gait)
            end

            if CONFIG.EnableProceduralHeadTracking
                and (string.find(partName, "head") or string.find(motorName, "neck")) then
                local look = camera and camera.CFrame.LookVector or Vector3.new(0, 0, -1)
                local localLook = rigData.model:GetPivot():VectorToObjectSpace(look)
                local yaw = math.clamp(math.atan2(localLook.X, -localLook.Z), -0.62, 0.62)
                local pitch = math.clamp(math.asin(math.clamp(localLook.Y, -1, 1)), -0.28, 0.28)
                transform = transform * CFrame.Angles(pitch, yaw, 0)
            end
            motor.Transform = transform
        end
    end
end

--==============================================================
-- 9. PIVOT E POSIÇÃO SOBRE O TERRENO
--==============================================================
local function groundPivotForData(rigData, horizontalPosition, yaw, excludes)
    local groundY = raycastGround(horizontalPosition, excludes)
    local y = groundY - (rigData and rigData.bottomOffset or -2.8) + 0.03
    return CFrame.new(horizontalPosition.X, y, horizontalPosition.Z) * CFrame.Angles(0, yaw, 0)
end

local function getCurrentPivot()
    if activeRig and activeRig.model and activeRig.model.Parent then
        return activeRig.model:GetPivot()
    end
    if humanRoot and humanRoot.Parent then
        return humanRoot.CFrame
    end
    return CFrame.new(0, 5, 0)
end

local function getExcludeList()
    local list = {}
    if character then
        table.insert(list, character)
    end
    if activeRig and activeRig.model then
        table.insert(list, activeRig.model)
    end
    return list
end

local function findWallBetween(startPosition, delta, excludes)
    if delta.Magnitude < 0.05 then
        return nil
    end
    local origin = startPosition + Vector3.new(0, CONFIG.WallRayHeight, 0)
    return Workspace:Raycast(origin, delta, makeRayParams(excludes))
end

--==============================================================
-- 10. SEQUÊNCIA COMPLETA DE TRANSFORMAÇÃO
--==============================================================
local function resetSourceEffects(rigData)
    if not rigData or not rigData.effects then
        return
    end
    for _, emitter in ipairs(rigData.effects.emitters or {}) do
        if emitter and emitter.Parent then
            emitter.Enabled = false
        end
    end
    for _, trail in ipairs(rigData.effects.trails or {}) do
        if trail and trail.Parent then
            trail.Enabled = false
        end
    end
    if rigData.effects.selection then
        rigData.effects.selection.Visible = false
    end
end

local function destroyRig(rigData)
    if not rigData then
        return
    end
    stopRigAnimations(rigData)
    resetSourceEffects(rigData)
    if rigData.model and rigData.model.Parent then
        rigData.model:Destroy()
    end
end

local function setCameraSubjectForRig(rigData)
    camera = Workspace.CurrentCamera or camera
    if not camera or not rigData then
        return
    end
    if rigData.kind == STATE_MECHA and rigData.humanoid then
        camera.CameraSubject = rigData.humanoid
    elseif rigData.root then
        camera.CameraSubject = rigData.root
    end
end

local function startTransitionEffects(rigData)
    local root = rigData and rigData.root
    if not root and humanRoot then
        root = humanRoot
    end
    playSound("Sound_Transform_Start", root, false)
    emitTransformEffects(rigData, 22)
    if rigData and rigData.effects and rigData.effects.selection then
        rigData.effects.selection.Visible = true
    end
end

local function finishTransitionEffects(rigData)
    local root = rigData and rigData.root or humanRoot
    playSound("Sound_Transform_End", root, false)
    emitTransformEffects(rigData, 30)
    if rigData and rigData.effects and rigData.effects.selection then
        rigData.effects.selection.Visible = false
    end
    if rigData and rigData.effects and rigData.effects.light then
        rigData.effects.light.Brightness = 2.2
    end
end

local function transitionTo(targetState)
    if isBusy or targetState == currentState then
        return
    end
    if targetState ~= STATE_HUMAN and targetState ~= STATE_MECHA and targetState ~= STATE_VEHICLE then
        return
    end

    if targetState == STATE_MECHA and not resolveTemplate(CONFIG.MechaTemplateName) then
        setStatus("MECHA AUSENTE", "CRIE ReplicatedStorage.MechaRig", CONFIG.WarningColor)
        return
    end
    if targetState == STATE_VEHICLE and not resolveTemplate(CONFIG.VehicleTemplateName) then
        setStatus("VEÍCULO AUSENTE", "CRIE ReplicatedStorage.VehicleRig", CONFIG.WarningColor)
        return
    end
    if currentState == STATE_HUMAN and targetState ~= STATE_HUMAN then
        if not suspendHuman() then
            return
        end
    end

    isBusy = true
    currentState = STATE_TRANSFORMING
    updateButtons()
    progressBack.Visible = true
    progressFill.Size = UDim2.fromScale(0, 1)
    setStatus("SISTEMA DE TRANSFORMAÇÃO ATIVADO...", "SINCRONIZANDO PLACAS // 0%", CONFIG.AccentColor)

    local sourceRig = activeRig
    local isReturningHuman = targetState == STATE_HUMAN
    local oldPivot = getCurrentPivot()
    local facingYaw = getYaw(oldPivot)
    local horizontalPosition = Vector3.new(oldPivot.Position.X, oldPivot.Position.Y, oldPivot.Position.Z)
    local targetRig = nil
    local targetPivot = oldPivot

    if not isReturningHuman then
        local template = resolveTemplate(targetState == STATE_MECHA and CONFIG.MechaTemplateName or CONFIG.VehicleTemplateName)
        local dummyPivot = template:GetPivot()
        local bottomOffset = calculateBottomOffset(template)
        local groundY = raycastGround(horizontalPosition, getExcludeList())
        local pivotY = groundY - bottomOffset + 0.03
        targetPivot = CFrame.new(horizontalPosition.X, pivotY, horizontalPosition.Z) * CFrame.Angles(0, facingYaw, 0)
        if targetState == STATE_MECHA then
            local err
            targetRig, err = prepareMechaModel(targetPivot, true)
            if not targetRig then
                warn("[Transformer Engine] " .. tostring(err))
                setStatus("FALHA AO CRIAR ROBÔ", tostring(err), CONFIG.WarningColor)
                isBusy = false
                currentState = sourceRig and sourceRig.kind or STATE_HUMAN
                progressBack.Visible = false
                updateButtons()
                if humanCache and humanCache.suspended and not sourceRig then
                    restoreHuman(oldPivot)
                end
                return
            end
        else
            local err
            targetRig, err = prepareVehicleModel(targetPivot, true)
            if not targetRig then
                warn("[Transformer Engine] " .. tostring(err))
                setStatus("FALHA AO CRIAR VEÍCULO", tostring(err), CONFIG.WarningColor)
                isBusy = false
                currentState = sourceRig and sourceRig.kind or STATE_HUMAN
                progressBack.Visible = false
                updateButtons()
                if humanCache and humanCache.suspended and not sourceRig then
                    restoreHuman(oldPivot)
                end
                return
            end
        end
        targetRig.pivot = targetPivot
        setExplodedPose(targetRig)
        setCameraSubjectForRig(targetRig)
        tweenCameraFOV(targetState == STATE_MECHA and CONFIG.MechaFOV or CONFIG.VehicleFOV, 0.7)
        if targetState == STATE_MECHA then
            loadAuthoredAnimations(targetRig)
        end
    else
        -- Prepara o avatar humano escondido para ser remontado na posição atual da máquina.
        if humanCache then
            local groundY = raycastGround(horizontalPosition, getExcludeList())
            local humanY = groundY - humanCache.rootBottomOffset + 0.03
            targetPivot = CFrame.new(horizontalPosition.X, humanY, horizontalPosition.Z) * CFrame.Angles(0, facingYaw, 0)
            placeHumanFolded(targetPivot)
        end
        camera = Workspace.CurrentCamera or camera
        if camera and humanCache then
            camera.CameraSubject = humanCache.humanoid
        end
        tweenCameraFOV(CONFIG.HumanFOV, 0.7)
    end

    if sourceRig and sourceRig.kind == STATE_MECHA then
        playTransformationTrack(sourceRig)
    end
    if targetRig and targetRig.kind == STATE_MECHA then
        playTransformationTrack(targetRig)
    end
    startTransitionEffects(targetRig or sourceRig)
    local duration = isReturningHuman and CONFIG.TransformHumanDuration or CONFIG.TransformDuration
    local startTime = os.clock()
    local midSoundPlayed = false

    while true do
        local elapsed = os.clock() - startTime
        local raw = clamp01(elapsed / duration)
        local collapseAlpha = easeOutCubic((raw - 0.10) / 0.82)
        local buildAlpha = easeOutBack((raw - 0.20) / 0.80)
        buildAlpha = math.clamp(buildAlpha, 0, 1.08)

        if sourceRig then
            animateSourceCollapse(sourceRig, collapseAlpha)
        elseif not isReturningHuman then
            animateHumanDisassembly(collapseAlpha, targetPivot)
        end

        if targetRig then
            animateTargetBuild(targetRig, buildAlpha)
        elseif isReturningHuman then
            animateHumanAssembly(buildAlpha, targetPivot)
        end

        progressFill.Size = UDim2.fromScale(raw, 1)
        statusLabel.Text = "SISTEMA DE TRANSFORMAÇÃO ATIVADO..."
        statusDetail.Text = "MONTAGEM MECÂNICA // " .. tostring(math.floor(raw * 100)) .. "%"
        blur.Size = math.sin(raw * math.pi) * 7

        if raw >= 0.52 and not midSoundPlayed then
            midSoundPlayed = true
            playSound("Sound_Transform_Mid", targetRig and targetRig.root or (sourceRig and sourceRig.root) or humanRoot, false)
            emitTransformEffects(targetRig or sourceRig, 18)
        end
        if raw >= 1 then
            break
        end
        RunService.RenderStepped:Wait()
    end

    blur.Size = 0
    if sourceRig then
        destroyRig(sourceRig)
    end
    activeRig = nil

    if isReturningHuman then
        local restored = restoreHuman(targetPivot)
        if not restored then
            setStatus("ERRO AO RESTAURAR HUMANO", "O AVATAR PODE TER SIDO RECRIADO", CONFIG.WarningColor)
        end
        currentState = STATE_HUMAN
        stopLoopSound()
        moveSpeedCurrent = 0
        vehicleSpeedCurrent = 0
        verticalVelocity = 0
        finishTransitionEffects(nil)
        setStatus("SISTEMA ONLINE // HUMANO", "CONTROLES ORIGINAIS RESTAURADOS", CONFIG.AccentColor)
    else
        activeRig = targetRig
        currentState = targetState
        if activeRig then
            activeRig.pivot = targetPivot
            setRigVisibility(activeRig, 0)
            if activeRig.kind == STATE_MECHA then
                for _, motorData in ipairs(activeRig.motors) do
                    if motorData.motor and motorData.motor.Parent then
                        motorData.motor.Transform = motorData.base
                    end
                end
                if activeRig.effects and activeRig.effects.trails then
                    for _, trail in ipairs(activeRig.effects.trails) do
                        trail.Enabled = false
                    end
                end
                changeMechaTrack(activeRig, "MechaIdle")
                stopLoopSound()
                setStatus("FORMA MECHA // ONLINE", "ATUADORES E REATOR ESTÁVEIS", CONFIG.AccentColor)
            else
                stopLoopSound()
                playSound("Sound_Vehicle_Engine", activeRig.root, true)
                setStatus("FORMA VEÍCULO // ONLINE", "TRAÇÃO E DIREÇÃO ATIVAS", CONFIG.AccentColor)
            end
            setCameraSubjectForRig(activeRig)
        end
        finishTransitionEffects(activeRig)
    end

    progressFill.Size = UDim2.fromScale(1, 1)
    task.delay(0.45, function()
        if not destroying and not isBusy then
            progressBack.Visible = false
            progressFill.Size = UDim2.fromScale(0, 1)
        end
    end)
    isBusy = false
    updateButtons()
end

--==============================================================
-- 11. MOVIMENTO E CONTROLE MOBILE
--==============================================================
local function moveMecha(dt, moveVector)
    if not activeRig or not activeRig.model or not activeRig.model.Parent then
        return
    end
    local pivot = activeRig.model:GetPivot()
    local position = pivot.Position
    local moveDirection = getCameraRelativeDirection(moveVector)
    local amount = math.clamp(Vector3.new(moveVector.X, 0, moveVector.Z).Magnitude, 0, 1)
    local shiftHeld = CONFIG.EnableKeyboardShortcuts and UserInputService:IsKeyDown(Enum.KeyCode.LeftShift)
    local targetSpeed = shiftHeld and CONFIG.MechaRunSpeed or CONFIG.MechaWalkSpeed
    if amount < 0.08 then
        targetSpeed = 0
    end
    moveSpeedCurrent = lerpNumber(moveSpeedCurrent, targetSpeed * amount, math.clamp(dt * 5.5, 0, 1))

    local nextPosition = position
    if moveDirection.Magnitude > 0.01 and moveSpeedCurrent > 0.05 then
        local delta = moveDirection * moveSpeedCurrent * dt
        local wall = findWallBetween(position, delta, getExcludeList())
        if not wall then
            nextPosition += delta
        end
    end

    local groundY = raycastGround(Vector3.new(nextPosition.X, position.Y, nextPosition.Z), getExcludeList())
    local groundedPivotY = groundY - activeRig.bottomOffset + 0.03
    if jumpRequested then
        if math.abs(position.Y - groundedPivotY) < 1.0 or position.Y < groundedPivotY + 0.25 then
            verticalVelocity = CONFIG.MechaJumpVelocity
            emitTransformEffects(activeRig, 12)
        end
        jumpRequested = false
    end
    verticalVelocity -= CONFIG.Gravity * dt
    local nextY = position.Y + verticalVelocity * dt
    if nextY <= groundedPivotY then
        nextY = groundedPivotY
        verticalVelocity = 0
    end

    local desiredCF
    if moveDirection.Magnitude > 0.05 then
        desiredCF = CFrame.lookAt(Vector3.new(nextPosition.X, nextY, nextPosition.Z), Vector3.new(nextPosition.X, nextY, nextPosition.Z) + moveDirection)
    else
        desiredCF = CFrame.new(nextPosition.X, nextY, nextPosition.Z) * CFrame.Angles(0, getYaw(pivot), 0)
    end
    activeRig.model:PivotTo(desiredCF)
    activeRig.pivot = desiredCF

    local walking = amount > 0.08
    local running = walking and (shiftHeld or amount > 0.88)
    local trackName = "MechaIdle"
    if verticalVelocity > 4 and activeRig.tracks.MechaJump then
        trackName = "MechaJump"
    elseif running then
        trackName = "MechaRun"
    elseif walking then
        trackName = "MechaWalk"
    end
    if not changeMechaTrack(activeRig, trackName) then
        applyProceduralPose(activeRig, sessionClock, amount, running)
    end

    footstepClock += dt
    if walking and footstepClock > (running and 0.36 or 0.56) then
        footstepClock = 0
        playSound("Sound_Footstep_Mecha", activeRig.root, false)
        if activeRig.effects and activeRig.effects.emitters then
            for _, emitter in ipairs(activeRig.effects.emitters) do
                if emitter and string.find(string.lower(emitter.Name), "smoke") then
                    safeCall(function() emitter:Emit(5) end)
                end
            end
        end
    end
    if activeRig.effects and activeRig.effects.light then
        activeRig.effects.light.Brightness = 1.45 + math.sin(sessionClock * 3.1) * 0.45
    end
    if activeRig.effects and activeRig.effects.trails then
        for _, trail in ipairs(activeRig.effects.trails) do
            trail.Enabled = running
        end
    end
end

local function moveVehicle(dt, moveVector)
    if not activeRig or not activeRig.model or not activeRig.model.Parent then
        return
    end
    local pivot = activeRig.model:GetPivot()
    local position = pivot.Position
    local forwardInput = math.clamp(-moveVector.Z, -1, 1)
    local steerInput = math.clamp(moveVector.X, -1, 1)
    local yaw = getYaw(pivot)

    local turnScale = math.abs(forwardInput) > 0.05 and math.sign(forwardInput) or 0.35
    local turnRate = math.abs(forwardInput) > 0.05 and CONFIG.VehicleTurnSpeed or CONFIG.VehicleIdleTurnSpeed
    yaw -= steerInput * turnRate * turnScale * dt

    local targetSpeed
    if forwardInput >= 0 then
        targetSpeed = CONFIG.VehicleMaxSpeed * forwardInput
    else
        targetSpeed = CONFIG.VehicleReverseSpeed * forwardInput
    end
    local accel = math.abs(targetSpeed) < math.abs(vehicleSpeedCurrent) and CONFIG.VehicleBrake or CONFIG.VehicleAcceleration
    vehicleSpeedCurrent = lerpNumber(vehicleSpeedCurrent, targetSpeed, math.clamp(dt * accel / 10, 0, 1))

    local headingCF = CFrame.new(position) * CFrame.Angles(0, yaw, 0)
    local delta = headingCF.LookVector * vehicleSpeedCurrent * dt
    local nextPosition = position
    if math.abs(vehicleSpeedCurrent) > 0.03 then
        local wall = findWallBetween(position, delta, getExcludeList())
        if not wall then
            nextPosition += delta
        else
            vehicleSpeedCurrent = 0
        end
    end

    local groundY = raycastGround(Vector3.new(nextPosition.X, position.Y, nextPosition.Z), getExcludeList())
    local nextY = groundY - activeRig.bottomOffset + 0.03
    local newPivot = CFrame.new(nextPosition.X, nextY, nextPosition.Z) * CFrame.Angles(0, yaw, 0)
    activeRig.model:PivotTo(newPivot)
    activeRig.pivot = newPivot

    -- Giro visual das rodas; os nomes aceitos incluem Wheel/Tire.
    local rotation = sessionClock * (vehicleSpeedCurrent / 2.2)
    for _, wheelData in ipairs(activeRig.wheels or {}) do
        local part = wheelData.part
        if part and part.Parent then
            part.CFrame = newPivot * wheelData.localCF * CFrame.Angles(rotation, 0, 0)
        end
    end
    if activeRig.effects and activeRig.effects.light then
        activeRig.effects.light.Brightness = 0.8 + math.clamp(math.abs(vehicleSpeedCurrent) / CONFIG.VehicleMaxSpeed, 0, 1) * 0.8
    end
end

--==============================================================
-- 12. ENTRADAS: BOTÕES, TECLADO E SALTO
--==============================================================
transformButton.Activated:Connect(function()
    if currentState == STATE_HUMAN then
        transitionTo(STATE_MECHA)
    elseif currentState == STATE_MECHA then
        transitionTo(STATE_VEHICLE)
    elseif currentState == STATE_VEHICLE then
        transitionTo(STATE_MECHA)
    end
end)

humanButton.Activated:Connect(function()
    transitionTo(STATE_HUMAN)
end)

jumpButton.Activated:Connect(function()
    if isBusy then
        return
    end
    if currentState == STATE_MECHA then
        jumpRequested = true
    elseif currentState == STATE_VEHICLE then
        -- Impulso visual curto para o veículo; não substitui uma física de suspensão.
        local attach = activeRig and activeRig.effects and activeRig.effects.emitters
        if attach then
            emitTransformEffects(activeRig, 16)
        end
        vehicleSpeedCurrent = math.min(vehicleSpeedCurrent + 8, CONFIG.VehicleMaxSpeed)
    end
end)

if CONFIG.EnableKeyboardShortcuts then
    UserInputService.InputBegan:Connect(function(input, processed)
        if processed or isBusy then
            return
        end
        if input.KeyCode == Enum.KeyCode.T then
            if currentState == STATE_HUMAN then
                transitionTo(STATE_MECHA)
            elseif currentState == STATE_MECHA then
                transitionTo(STATE_VEHICLE)
            elseif currentState == STATE_VEHICLE then
                transitionTo(STATE_MECHA)
            end
        elseif input.KeyCode == Enum.KeyCode.H then
            transitionTo(STATE_HUMAN)
        elseif input.KeyCode == Enum.KeyCode.Space and (currentState == STATE_MECHA or currentState == STATE_VEHICLE) then
            jumpRequested = true
        end
    end)
end

--==============================================================
-- 13. CÂMERA E LOOP PRINCIPAL
--==============================================================
RunService:BindToRenderStep("TransformerEngineCameraFX", Enum.RenderPriority.Camera.Value + 1, function(dt)
    if destroying or not CONFIG.EnableCameraShake or isBusy or not activeRig then
        return
    end
    camera = Workspace.CurrentCamera or camera
    if not camera then
        return
    end
    local speed = currentState == STATE_VEHICLE and math.abs(vehicleSpeedCurrent) or moveSpeedCurrent
    local intensity = math.clamp(speed / (currentState == STATE_VEHICLE and CONFIG.VehicleMaxSpeed or CONFIG.MechaRunSpeed), 0, 1)
    if intensity > 0.04 then
        local x = math.sin(sessionClock * CONFIG.CameraShakeFrequency) * CONFIG.CameraShake * intensity * 0.18
        local y = math.abs(math.cos(sessionClock * CONFIG.CameraShakeFrequency * 1.5)) * CONFIG.CameraShake * intensity * 0.22
        camera.CFrame = camera.CFrame * CFrame.new(x, y, 0)
    end
end)

RunService.RenderStepped:Connect(function(dt)
    if destroying then
        return
    end
    sessionClock += dt
    if isBusy then
        return
    end
    if currentState ~= STATE_MECHA and currentState ~= STATE_VEHICLE then
        return
    end
    local moveVector = getMoveVector()
    if currentState == STATE_MECHA then
        moveMecha(dt, moveVector)
    elseif currentState == STATE_VEHICLE then
        moveVehicle(dt, moveVector)
    end
end)

--==============================================================
-- 14. CICLO DE VIDA E DIAGNÓSTICO VISÍVEL
--==============================================================
player.CharacterAdded:Connect(function(newCharacter)
    -- Se o personagem reaparecer enquanto estiver na forma humana, atualiza as referências.
    character = newCharacter
    if currentState == STATE_HUMAN and not isBusy then
        task.defer(function()
            local newHumanoid = newCharacter:WaitForChild("Humanoid", 10)
            local newRoot = newCharacter:WaitForChild("HumanoidRootPart", 10)
            if newHumanoid and newRoot and currentState == STATE_HUMAN then
                humanoid = newHumanoid
                humanRoot = newRoot
                humanCache = nil
                camera = Workspace.CurrentCamera or camera
                if camera then
                    camera.CameraSubject = humanoid
                end
                setStatus("SISTEMA ONLINE // HUMANO", "NOVO AVATAR DETECTADO", CONFIG.AccentColor)
            end
        end)
    end
end)

script.Destroying:Connect(function()
    destroying = true
    stopLoopSound()
    RunService:UnbindFromRenderStep("TransformerEngineCameraFX")
    if activeRig then
        destroyRig(activeRig)
        activeRig = nil
    end
    if humanCache and humanCache.suspended then
        -- Melhor esforço para não deixar o avatar invisível caso o script seja removido.
        restoreHuman(getCurrentPivot())
    end
    if blur and blur.Parent then
        blur.Size = 0
    end
    if screenGui and screenGui.Parent then
        screenGui:Destroy()
    end
end)

-- Sinal de inicialização visível para diagnóstico no celular.
local mechaTemplateExists = resolveTemplate(CONFIG.MechaTemplateName) ~= nil
local vehicleTemplateExists = resolveTemplate(CONFIG.VehicleTemplateName) ~= nil
if mechaTemplateExists and vehicleTemplateExists then
    setStatus("SISTEMA ONLINE // HUMANO", "ROBÔ E VEÍCULO DETECTADOS", CONFIG.AccentColor)
elseif not mechaTemplateExists and not vehicleTemplateExists then
    setStatus("AGUARDANDO MODELOS", "FALTAM MechaRig E VehicleRig", CONFIG.WarningColor)
elseif not mechaTemplateExists then
    setStatus("AGUARDANDO MODELO", "FALTA ReplicatedStorage.MechaRig", CONFIG.WarningColor)
else
    setStatus("AGUARDANDO MODELO", "FALTA ReplicatedStorage.VehicleRig", CONFIG.WarningColor)
end
updateButtons()
