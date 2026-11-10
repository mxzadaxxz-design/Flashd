-- FLASH para execução no cliente (Xeno).
-- Cole TODO este arquivo no executor, já dentro da experiência, e execute.
-- + / -: ajuste de 50; clique no número para digitar; Q: lento/retomar.
-- Limite: 16 a 3000 studs/s. Q mantém movimento lento e livre, com velocidade alta salva.
-- Modo cinematográfico local: não altera o tempo do servidor ou de outros jogadores.
-- Efeitos são locais. O servidor pode corrigir ou impedir a movimentação.
-- × encerra e restaura a velocidade; executar novamente substitui esta instância.

local Players = game:GetService('Players')
local UIS = game:GetService('UserInputService')
local Run = game:GetService('RunService')
local Tween = game:GetService('TweenService')
local player = Players.LocalPlayer
local playerGui = player:WaitForChild('PlayerGui')
local previous = playerGui:FindFirstChild('FlashUI')
if previous then
    local stop = previous:FindFirstChild('FlashStop')
    if stop then stop:Fire() else previous:Destroy() end
end
local connections, trailObjects, trails = {}, {}, {}
local alive = true
local normalSpeed = 16
local function connect(signal, callback)
    local connection = signal:Connect(callback)
    table.insert(connections, connection)
    return connection
end
local speed, enabled, slow = 200, false, false
-- Ritmo no modo cinematográfico; independente da velocidade salva.
local SLOW_SPEED = 24
local SLOW_ANIMATION_RATE = 0.35
local slowedTracks = {}
local function restoreAnimations()
    for track, rate in pairs(slowedTracks) do
        pcall(function() track:AdjustSpeed(rate) end)
    end
    table.clear(slowedTracks)
end
local char, hum, root
local orange = Color3.fromRGB(255, 175, 55)
local function make(class, props, parent)
    local obj = Instance.new(class)
    for k, v in pairs(props) do obj[k] = v end
    obj.Parent = parent
    return obj
end
local gui = make('ScreenGui', {Name = 'FlashUI', ResetOnSpawn = false, IgnoreGuiInset = false, ZIndexBehavior = Enum.ZIndexBehavior.Sibling}, player:WaitForChild('PlayerGui'))
local panel = make('Frame', {AnchorPoint = Vector2.new(0, 1), Position = UDim2.new(0, 20, 1, -24), Size = UDim2.fromOffset(300, 244), BackgroundColor3 = Color3.fromRGB(12, 12, 15), BackgroundTransparency = 0.15, BorderSizePixel = 0}, gui)
make('UICorner', {CornerRadius = UDim.new(0, 16)}, panel)
make('UIStroke', {Color = Color3.fromRGB(90, 90, 95), Transparency = 0.35, Thickness = 1}, panel)
local scale = make('UIScale', {Scale = 1}, panel)
local function resize()
    local camera = workspace.CurrentCamera
    if camera then scale.Scale = math.clamp(math.min(camera.ViewportSize.X / 360, camera.ViewportSize.Y / 360), 0.55, 1) end
end
local function label(text, x, y, w, h, size, color)
    return make('TextLabel', {Text = text, Position = UDim2.fromOffset(x, y), Size = UDim2.fromOffset(w, h), BackgroundTransparency = 1, Font = Enum.Font.GothamBold, TextSize = size, TextColor3 = color or Color3.new(1, 1, 1), TextXAlignment = Enum.TextXAlignment.Left}, panel)
end
label('FLASH', 20, 14, 170, 28, 24)
label('SPEED FORCE', 21, 43, 170, 17, 10, Color3.fromRGB(160, 160, 170))
local status = label('DESLIGADO', 185, 20, 100, 25, 10, orange)
local function button(text, x, y, w, h)
    local b = make('TextButton', {Text = text, Position = UDim2.fromOffset(x, y), Size = UDim2.fromOffset(w, h), BackgroundColor3 = Color3.fromRGB(35, 35, 41), BorderSizePixel = 0, TextColor3 = Color3.new(1, 1, 1), Font = Enum.Font.GothamBold, TextSize = 16, AutoButtonColor = true}, panel)
    make('UICorner', {CornerRadius = UDim.new(0, 10)}, b)
    return b
end
local minus = button('−', 20, 77, 45, 48)
local plus = button('+', 235, 77, 45, 48)
local number = make('TextBox', {Text = '200', Position = UDim2.fromOffset(74, 77), Size = UDim2.fromOffset(152, 48), BackgroundTransparency = 1, TextColor3 = Color3.new(1, 1, 1), TextSize = 32, Font = Enum.Font.GothamBold, ClearTextOnFocus = false}, panel)
local actual = label('VELOCIDADE · STUDS/S', 74, 126, 210, 16, 9, Color3.fromRGB(150, 150, 160))
local bar = make('Frame', {Position = UDim2.fromOffset(20, 150), Size = UDim2.fromOffset(260, 3), BackgroundColor3 = Color3.fromRGB(50, 50, 55), BorderSizePixel = 0}, panel)
local fill = make('Frame', {Size = UDim2.fromScale(200 / 3000, 1), BackgroundColor3 = orange, BorderSizePixel = 0}, bar)
local power = button('ATIVAR', 20, 168, 125, 38)
local mode = button('Q · LENTO', 155, 168, 125, 38)
label('+ / −  ajustar  •  Q  alternar modo', 21, 215, 260, 15, 10, Color3.fromRGB(150, 150, 160))
local function refresh()
    if not number:IsFocused() then number.Text = tostring(speed) end
    status.Text = enabled and (slow and 'MODO LENTO' or 'ATIVADO') or 'DESLIGADO'
    status.TextColor3 = enabled and orange or Color3.fromRGB(145, 145, 155)
    power.Text = enabled and 'DESATIVAR' or 'ATIVAR'
    mode.Text = slow and 'Q · RETOMAR' or 'Q · LENTO'
    Tween:Create(fill, TweenInfo.new(0.15), {Size = UDim2.fromScale(speed / 3000, 1)}):Play()
end
local function setSpeed(value)
    speed = math.clamp(math.floor(value), 16, 3000)
    refresh()
end
local function brake(limit)
    if not root then return end
    local v = root.AssemblyLinearVelocity
    local flat = Vector3.new(v.X, 0, v.Z)
    if flat.Magnitude > limit then flat = flat.Unit * limit end
    root.AssemblyLinearVelocity = Vector3.new(flat.X, v.Y, flat.Z)
end
local function togglePower()
    if not hum or hum.Health <= 0 then return end
    if not enabled then normalSpeed = hum.WalkSpeed end
    enabled = not enabled; slow = false
    restoreAnimations()
    hum.WalkSpeed = enabled and speed or normalSpeed
    if not enabled then brake(normalSpeed) end
    refresh()
end
local function toggleSlow()
    if not enabled then return end
    slow = not slow
    if slow then brake(math.min(SLOW_SPEED, speed * 0.75)) else restoreAnimations() end
    if hum then hum.WalkSpeed = slow and math.min(SLOW_SPEED, speed * 0.75) or speed end
    refresh()
end
connect(minus.Activated, function() setSpeed(speed - 50) end)
connect(plus.Activated, function() setSpeed(speed + 50) end)
connect(power.Activated, togglePower)
connect(mode.Activated, toggleSlow)
connect(number.FocusLost, function()
    local value = tonumber(number.Text)
    if value and value == value and math.abs(value) < math.huge then setSpeed(value) end
    number.Text = tostring(speed)
end)
connect(UIS.InputBegan, function(input, processed)
    if processed or UIS:GetFocusedTextBox() then return end
    local key = input.KeyCode
    if key == Enum.KeyCode.Q then toggleSlow()
    elseif key == Enum.KeyCode.Equals or key == Enum.KeyCode.KeypadPlus then setSpeed(speed + 50)
    elseif key == Enum.KeyCode.Minus or key == Enum.KeyCode.KeypadMinus then setSpeed(speed - 50) end
end)
local params = RaycastParams.new()
params.FilterType = Enum.RaycastFilterType.Exclude
params.RespectCanCollide = true
local effectFolder = make('Folder', {Name = 'FlashLocalEffects_' .. player.UserId}, workspace)
local bolts = {}
for i = 1, 24 do
    bolts[i] = make('Part', {Name = 'Lightning', Anchored = true, CanCollide = false, CanTouch = false, CanQuery = false, CastShadow = false, Material = Enum.Material.Neon, Color = orange, Transparency = 1, Size = Vector3.new(0.07, 0.07, 1)}, effectFolder)
end
local cameraBase, lastCamera, ownFov = 70, nil, nil
local function restoreCamera()
    if lastCamera and ownFov and math.abs(lastCamera.FieldOfView - ownFov) < 0.1 then
        lastCamera.FieldOfView = cameraBase
    end
    ownFov = nil
end
local function bind(character)
    restoreAnimations()
    char, hum, root = nil, nil, nil
    enabled, slow = false, false
    restoreCamera(); refresh()
    local h = character:WaitForChild('Humanoid', 10)
    local r = character:WaitForChild('HumanoidRootPart', 10)
    if not alive or not h or not r or player.Character ~= character then return end
    char, hum, root = character, h, r
    params.FilterDescendantsInstances = {character, effectFolder}
    normalSpeed = h.WalkSpeed
    for _, object in ipairs(trailObjects) do object:Destroy() end
    trailObjects, trails = {}, {}
    for _, name in ipairs({'LeftHand', 'RightHand', 'LeftFoot', 'RightFoot', 'Left Arm', 'Right Arm', 'Left Leg', 'Right Leg'}) do
        local part = character:FindFirstChild(name)
        if part and part:IsA('BasePart') then
            local a = make('Attachment', {Name = 'FlashTrailTop', Position = Vector3.new(0, 0.25, 0)}, part)
            local b = make('Attachment', {Name = 'FlashTrailBottom', Position = Vector3.new(0, -0.25, 0)}, part)
            local trail = make('Trail', {
                Name = 'FlashTrail', Attachment0 = a, Attachment1 = b,
                Color = ColorSequence.new(Color3.fromRGB(255, 235, 130), Color3.fromRGB(255, 75, 10)),
                Transparency = NumberSequence.new({NumberSequenceKeypoint.new(0, 0.12), NumberSequenceKeypoint.new(1, 1)}),
                Lifetime = 0.22, LightEmission = 1, MinLength = 0.1, Enabled = false
            }, part)
            table.insert(trails, trail)
            table.insert(trailObjects, trail)
            table.insert(trailObjects, a)
            table.insert(trailObjects, b)
        end
    end
    connect(h.Died, function() enabled = false; slow = false; restoreCamera(); refresh() end)
    refresh()
end
connect(player.CharacterAdded, bind)
if player.Character then task.spawn(bind, player.Character) end
-- Velocidade horizontal imediata: o modo lento freia mesmo no ar.
-- Blockcast reduz atravessamento de paredes; não substitui validação de movimento do servidor.
connect(Run.PreSimulation, function(dt)
    if enabled and hum and hum.Sit then hum.WalkSpeed = normalSpeed end
    if not enabled or not hum or not root or hum.Health <= 0 or hum.Sit or root.Anchored then return end
    local target = slow and math.min(SLOW_SPEED, speed * 0.75) or speed
    hum.WalkSpeed = target
    local direction = hum.MoveDirection
    local flat = Vector3.new(direction.X, 0, direction.Z)
    if flat.Magnitude > 0.01 then flat = flat.Unit * target else flat = Vector3.zero end
    if flat.Magnitude > 0 and dt > 0 then
        local distance = flat.Magnitude * dt
        local hit = workspace:Blockcast(root.CFrame, Vector3.new(1.6, 1.8, 1.6), flat.Unit * (distance + 0.6), params)
        if hit then flat = flat.Unit * math.min(target, math.max(0, hit.Distance - 0.6) / dt) end
    end
    local v = root.AssemblyLinearVelocity
    root.AssemblyLinearVelocity = Vector3.new(flat.X, v.Y, flat.Z)
end)
local rng = Random.new()
local elapsed = 0
connect(Run.RenderStepped, function(dt)
    resize()
    local running = enabled and root and hum and hum.Health > 0 and not hum.Sit and hum.MoveDirection.Magnitude > 0.05
    for _, trail in ipairs(trails) do
        trail.Enabled = running == true
        trail.Lifetime = slow and 0.8 or 0.22
    end
    if enabled and slow and hum and hum.Health > 0 and not hum.Sit then
        local animator = hum:FindFirstChildOfClass('Animator')
        if animator then
            for _, track in ipairs(animator:GetPlayingAnimationTracks()) do
                local name = string.lower(track.Name)
                if string.find(name, 'walk', 1, true) or string.find(name, 'run', 1, true) then
                    if slowedTracks[track] == nil then slowedTracks[track] = track.Speed end
                    track:AdjustSpeed(SLOW_ANIMATION_RATE)
                end
            end
        end
    elseif next(slowedTracks) then restoreAnimations() end
    local camera = workspace.CurrentCamera
    if camera ~= lastCamera then
        restoreCamera(); lastCamera = camera
        if camera then cameraBase = camera.FieldOfView end
    end
    if camera then
        if running then
            if not ownFov then cameraBase = camera.FieldOfView end
            local goal = slow and math.max(50, cameraBase - 8) or math.min(100, cameraBase + 20)
            ownFov = camera.FieldOfView + (goal - camera.FieldOfView) * (1 - math.exp(-dt * 7))
            camera.FieldOfView = ownFov
        elseif ownFov then
            ownFov = camera.FieldOfView + (cameraBase - camera.FieldOfView) * (1 - math.exp(-dt * 9))
            camera.FieldOfView = ownFov
            if math.abs(ownFov - cameraBase) < 0.05 then restoreCamera() end
        end
    end
    elapsed = elapsed + dt
    if elapsed < (slow and 0.18 or 0.045) then return end
    elapsed = 0
    local measured = root and Vector3.new(root.AssemblyLinearVelocity.X, 0, root.AssemblyLinearVelocity.Z).Magnitude or 0
    actual.Text = string.format(slow and 'CÂMERA LENTA · %d STUDS/S' or 'ATUAL %d · STUDS/S', math.floor(measured))
    if not running then
        for _, p in ipairs(bolts) do p.Transparency = 1 end
        return
    end
    local direction = hum.MoveDirection.Unit
    local side = direction:Cross(Vector3.yAxis)
    for branch = 0, 3 do
        local origin = root.Position + side * rng:NextNumber(-1.2, 1.2) + Vector3.new(0, rng:NextNumber(-1.8, 1.5), 0)
        for segment = 1, 6 do
            local nextPoint = origin - direction * rng:NextNumber(0.8, 2.2) + side * rng:NextNumber(-0.6, 0.6) + Vector3.new(0, rng:NextNumber(-0.45, 0.45), 0)
            local p = bolts[branch * 6 + segment]
            p.Size = Vector3.new(0.075, 0.075, (nextPoint - origin).Magnitude)
            p.CFrame = CFrame.lookAt((origin + nextPoint) / 2, nextPoint)
            p.Transparency = segment / 12
            origin = nextPoint
        end
    end
end)
refresh()

local stopEvent = make('BindableEvent', {Name = 'FlashStop'}, gui)
local close = button('×', 254, 43, 26, 23)
local function cleanup()
    if not alive then return end
    alive = false
    restoreAnimations()
    enabled, slow = false, false
    for _, connection in ipairs(connections) do connection:Disconnect() end
    if hum and hum.Parent and hum.Health > 0 then hum.WalkSpeed = normalSpeed end
    if root and root.Parent then brake(normalSpeed) end
    restoreCamera()
    for _, object in ipairs(trailObjects) do object:Destroy() end
    effectFolder:Destroy()
    gui:Destroy()
end
connect(stopEvent.Event, cleanup)
connect(close.Activated, cleanup)
print('[FLASH] Carregado: ative no painel. + / - ajustam; Q alterna modo lento; × encerra.')
