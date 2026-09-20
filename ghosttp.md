-- Ghost TP: LocalScript para o seu próprio jogo.
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")
if playerGui:FindFirstChild("GhostTP") then return end

local SPEED = 100
local TP_FREQUENCY = 20 -- Limite seguro por segundo.
local PURPLE = Color3.fromRGB(158, 75, 255)
local savedCFrame = nil
local mode = nil
local activeRoot = nil
local originalAnchored = false
local elapsed = 0
local deathConnection = nil
local characterVersion = 0
local connections = {}

local function create(className, properties, parent)
	local object = Instance.new(className)
	for key, value in pairs(properties) do object[key] = value end
	object.Parent = parent
	return object
end

local function rounded(object, radius)
	create("UICorner", {CornerRadius = UDim.new(0, radius)}, object)
end

local gui = create("ScreenGui", {
	Name = "GhostTP", ResetOnSpawn = false,
	ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
	ScreenInsets = Enum.ScreenInsets.DeviceSafeInsets,
}, playerGui)

local launcher = create("TextButton", {
	Name = "Launcher", Size = UDim2.fromOffset(66, 66),
	Position = UDim2.new(0, 16, 0.5, -33),
	BackgroundColor3 = PURPLE, Text = "Ghost\nTP",
	TextColor3 = Color3.new(1, 1, 1), TextSize = 17,
	Font = Enum.Font.GothamBold, AutoButtonColor = false,
}, gui)
rounded(launcher, 16)
create("UIStroke", {Color = Color3.fromRGB(205, 160, 255), Thickness = 2}, launcher)

local panel = create("Frame", {
	Name = "Menu", Size = UDim2.fromOffset(310, 394),
	AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5),
	BackgroundColor3 = Color3.fromRGB(12, 10, 18),
	Visible = false, Active = true,
}, gui)
rounded(panel, 18)
create("UIStroke", {Color = PURPLE, Thickness = 3}, panel)
local scale = create("UIScale", {Scale = 1}, panel)

local function resize()
	local camera = workspace.CurrentCamera
	if camera then
		local size = camera.ViewportSize
		scale.Scale = math.clamp(math.min((size.X - 28) / 310, (size.Y - 70) / 394), 0.35, 1)
	end
end

local header = create("TextLabel", {
	Size = UDim2.new(1, -62, 0, 60), Position = UDim2.fromOffset(18, 0),
	BackgroundTransparency = 1, Text = "GHOST TP",
	TextColor3 = Color3.new(1, 1, 1), Font = Enum.Font.GothamBold,
	TextSize = 22, TextXAlignment = Enum.TextXAlignment.Left, Active = true,
}, panel)
local close = create("TextButton", {
	Size = UDim2.fromOffset(34, 34), Position = UDim2.new(1, -48, 0, 13),
	BackgroundColor3 = Color3.fromRGB(43, 25, 61), Text = "X",
	TextColor3 = Color3.new(1, 1, 1), Font = Enum.Font.GothamBold, TextSize = 16,
}, panel)
rounded(close, 10)

local status = create("TextLabel", {
	Position = UDim2.fromOffset(18, 306), Size = UDim2.new(1, -36, 0, 68),
	BackgroundTransparency = 1, Text = "Salve uma posição para começar.",
	TextColor3 = Color3.fromRGB(203, 193, 219), TextSize = 14,
	Font = Enum.Font.Gotham, TextWrapped = true,
}, panel)

local function notify(text, warning)
	status.Text = text
	status.TextColor3 = warning and Color3.fromRGB(255, 179, 145) or Color3.fromRGB(210, 185, 255)
end

local function button(text, y)
	local b = create("TextButton", {
		Size = UDim2.new(1, -36, 0, 48), Position = UDim2.fromOffset(18, y),
		BackgroundColor3 = Color3.fromRGB(47, 26, 70), Text = text,
		TextColor3 = Color3.new(1, 1, 1), TextSize = 16,
		Font = Enum.Font.GothamMedium, AutoButtonColor = false,
	}, panel)
	rounded(b, 12)
	create("UIStroke", {Color = PURPLE, Transparency = 0.55}, b)
	b.MouseEnter:Connect(function()
		TweenService:Create(b, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(79, 39, 120)}):Play()
	end)
	b.MouseLeave:Connect(function()
		TweenService:Create(b, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(47, 26, 70)}):Play()
	end)
	return b
end

local saveButton = button("Salvar Posição", 70)
local tpButton = button("TP", 128)
local bugButton = button("Bug TP", 186)
local routeButton = button("Ir pelo Trajeto", 244)

local function getCharacter()
	local character = player.Character
	if not character then return nil, nil end
	local humanoid = character:FindFirstChildOfClass("Humanoid")
	local root = character:FindFirstChild("HumanoidRootPart")
	if not humanoid or humanoid.Health <= 0 or not root or not root:IsA("BasePart") then
		return nil, nil
	end
	return root, humanoid
end

local function stopMovement(message)
	if mode == "route" and activeRoot and activeRoot.Parent then
		activeRoot.Anchored = originalAnchored
		activeRoot.AssemblyLinearVelocity = Vector3.zero
		activeRoot.AssemblyAngularVelocity = Vector3.zero
	end
	mode = nil
	activeRoot = nil
	elapsed = 0
	bugButton.Text = "Bug TP"
	routeButton.Text = "Ir pelo Trajeto"
	if message then notify(message) end
end

local function ready()
	if not savedCFrame then
		notify("Salve uma posição primeiro!", true)
		return nil
	end
	local root, humanoid = getCharacter()
	if not root then
		notify("Personagem indisponível.", true)
		return nil
	end
	if humanoid.SeatPart then
		notify("Saia do assento antes de teleportar.", true)
		return nil
	end
	return root
end

local function teleport(root)
	root.CFrame = savedCFrame
	root.AssemblyLinearVelocity = Vector3.zero
	root.AssemblyAngularVelocity = Vector3.zero
end

saveButton.Activated:Connect(function()
	local root = getCharacter()
	if not root then notify("Personagem indisponível.", true) return end
	stopMovement()
	savedCFrame = root.CFrame
	notify("✓ Posição e orientação salvas!")
end)

tpButton.Activated:Connect(function()
	local root = ready()
	if not root then return end
	stopMovement()
	teleport(root)
	notify("Teletransporte concluído.")
end)

bugButton.Activated:Connect(function()
	if mode == "bug" then stopMovement("Bug TP interrompido.") return end
	local root = ready()
	if not root then return end
	stopMovement()
	activeRoot = root
	mode = "bug"
	bugButton.Text = "Parar Bug TP"
	teleport(root)
	notify("Retorno contínuo ativo: 20 vezes/s.")
end)

routeButton.Activated:Connect(function()
	if mode == "route" then stopMovement("Trajeto cancelado.") return end
	local root = ready()
	if not root then return end
	stopMovement()
	activeRoot = root
	originalAnchored = root.Anchored
	root.Anchored = true
	mode = "route"
	routeButton.Text = "Cancelar Trajeto"
	notify("Percorrendo o trajeto a 100 studs/s.")
end)

table.insert(connections, RunService.Heartbeat:Connect(function(dt)
	if not mode then return end
	local root = getCharacter()
	if not root or root ~= activeRoot then
		stopMovement("Movimento interrompido.")
		return
	end
	if mode == "bug" then
		elapsed += dt
		if elapsed >= 1 / TP_FREQUENCY then
			elapsed = 0 -- Não acumula milhares de chamadas por frame.
			teleport(root)
		end
	else
		local distance = (savedCFrame.Position - root.Position).Magnitude
		local step = SPEED * math.min(dt, 0.1)
		if distance <= step then
			teleport(root)
			stopMovement("Destino alcançado!")
		else
			root.CFrame = root.CFrame:Lerp(savedCFrame, step / distance)
		end
	end
end))

local function bindCharacter(character)
	characterVersion += 1
	local version = characterVersion
	stopMovement("Movimentos interrompidos; posição salva mantida.")
	if deathConnection then deathConnection:Disconnect() deathConnection = nil end
	task.spawn(function()
		local humanoid = character:WaitForChild("Humanoid", 10)
		if version ~= characterVersion or not humanoid or player.Character ~= character then return end
		deathConnection = humanoid.Died:Connect(function()
			stopMovement("Personagem morreu. Movimento encerrado.")
		end)
	end)
end

table.insert(connections, player.CharacterAdded:Connect(bindCharacter))
table.insert(connections, player.CharacterRemoving:Connect(function()
	characterVersion += 1
	stopMovement("Aguardando personagem...")
	if deathConnection then deathConnection:Disconnect() deathConnection = nil end
end))
if player.Character then bindCharacter(player.Character) end

-- Arrasto compatível com mouse e toque.
local dragged = false
local function draggable(handle, target)
	local inputStart, origin, touchInput
	local dragging = false
	handle.InputBegan:Connect(function(input)
		if input.UserInputType ~= Enum.UserInputType.MouseButton1 and input.UserInputType ~= Enum.UserInputType.Touch then return end
		dragging = true
		dragged = false
		inputStart = input.Position
		origin = target.Position
		touchInput = input
	end)
	table.insert(connections, UserInputService.InputChanged:Connect(function(input)
		if not dragging then return end
		if input ~= touchInput and not (touchInput.UserInputType == Enum.UserInputType.MouseButton1 and input.UserInputType == Enum.UserInputType.MouseMovement) then return end
		local delta = input.Position - inputStart
		if delta.Magnitude > 7 then dragged = true end
		if dragged then
			target.Position = UDim2.new(origin.X.Scale, origin.X.Offset + delta.X, origin.Y.Scale, origin.Y.Offset + delta.Y)
		end
	end))
	table.insert(connections, UserInputService.InputEnded:Connect(function(input)
		if input == touchInput then
			dragging = false
			task.delay(0.15, function() dragged = false end)
		end
	end))
end

draggable(launcher, launcher)
draggable(header, panel)
launcher.Activated:Connect(function()
	if dragged then return end
	resize()
	panel.Visible = not panel.Visible
end)
close.Activated:Connect(function() panel.Visible = false end)

local lastViewport = Vector2.zero
table.insert(connections, RunService.RenderStepped:Connect(function()
	local camera = workspace.CurrentCamera
	if camera and camera.ViewportSize ~= lastViewport then
		lastViewport = camera.ViewportSize
		resize()
	end
end))

gui.Destroying:Connect(function()
	characterVersion += 1
	stopMovement()
	if deathConnection then deathConnection:Disconnect() end
	for _, connection in ipairs(connections) do connection:Disconnect() end
end)
resize()
