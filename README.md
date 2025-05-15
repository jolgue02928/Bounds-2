-- what it can do just boring stuff

local TweenService = game:GetService("TweenService")
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TeleportService = game:GetService("TeleportService")

local screenGui = Instance.new("ScreenGui")
screenGui.Name = "BondUI"
screenGui.ResetOnSpawn = false
screenGui.Parent = cloneref(game:GetService("CoreGui"))

local bondLabel = Instance.new("TextLabel")
bondLabel.Size = UDim2.new(0.3, 0, 0.1, 0)
bondLabel.Position = UDim2.new(0.35, 0, 0.45, 0)
bondLabel.BackgroundTransparency = 1
bondLabel.TextScaled = true
bondLabel.Font = Enum.Font.SourceSansBold
bondLabel.TextColor3 = Color3.new(1, 1, 1)
bondLabel.TextStrokeTransparency = 0.5
bondLabel.Text = "Bond Collected: 0"
bondLabel.Parent = screenGui

local player = Players.LocalPlayer
local character = player.Character or player.CharacterAdded:Wait()
local root = character:WaitForChild("HumanoidRootPart")
local lockedY = root.Position.Y

local activateRemote = ReplicatedStorage:WaitForChild("Shared")
    :WaitForChild("Network")
    :WaitForChild("RemotePromise")
    :WaitForChild("Remotes")
    :WaitForChild("C_ActivateObject")

local function findNearestBond()
    local closest, shortestDist = nil, math.huge
    for _, item in ipairs(workspace.RuntimeItems:GetDescendants()) do
        if item:IsA("Model") and item.Name:lower() == "bond" then
            local primary = item.PrimaryPart or item:FindFirstChildWhichIsA("BasePart")
            if primary then
                local dist = (primary.Position - root.Position).Magnitude
                if dist < shortestDist then
                    shortestDist = dist
                    closest = item
                end
            end
        end
    end
    return closest
end

-- Lock Y-axis So You Didn't fall from tween
task.spawn(function()
    while true do
        if root and root.Parent then
            local pos = root.Position
            root.Velocity = Vector3.new(root.Velocity.X, 0, root.Velocity.Z)
            root.CFrame = CFrame.new(pos.X, lockedY, pos.Z)
        end
        task.wait()
    end
end)

local bondCount = 0  -- Useless?

-- Teleport to Bond and activate (loop until taken or timeout)
local function teleportTo(bond)
    local primary = bond.PrimaryPart or bond:FindFirstChildWhichIsA("BasePart")
    if not primary then return end

    root.CFrame = primary.CFrame + Vector3.new(0, 5, 0)

    local startTime = os.clock()
    while bond.Parent and os.clock() - startTime < 1 do -- 5 second timeout
        activateRemote:FireServer(bond)
        task.wait(0.1)
    end

    if not bond.Parent then
        bondCount += 1
        bondLabel.Text = "Bond Collected: " .. bondCount
        print("Bond taken! Total:", bondCount)
    else
        print("Failed to take bond within timeout.")
    end
end

local activeTween = nil
local travelTime = 1
local function tweenTo(pos)
    if activeTween then
        activeTween:Cancel()
    end

    local tweenInfo = TweenInfo.new(travelTime, Enum.EasingStyle.Linear)
    local goal = {CFrame = CFrame.new(pos)}
    activeTween = TweenService:Create(root, tweenInfo, goal)

    local finished = false
    local lastTweenStart = os.clock()

    local connection
    connection = activeTween.Completed:Connect(function()
        finished = true
        if connection then connection:Disconnect() end
    end)

    activeTween:Play()

    while not finished do
        local bond = findNearestBond()
        if bond then
            activeTween:Cancel()
            teleportTo(bond)
            return
        end

        if os.clock() - lastTweenStart > 5 and not finished then
            warn("Tween possibly stuck, retrying...")
            return tweenTo(pos)  -- Retry
        end

        task.wait(0.1)
    end
end

local layerSize = 2048
local halfSize = layerSize / 2
local xStart = -halfSize
local xEnd = halfSize
local y = -50
local zStart = 30000
local zEnd = -49872
local zStep = -layerSize

while true do
    local bond = findNearestBond()
    if bond then
        teleportTo(bond)
    else
        break
    end
    task.wait(0.05)
end

-- Zig-zag scan + bond glitches ass check
local z = zStart
local direction = 1

while (zStep < 0 and z >= zEnd) or (zStep > 0 and z <= zEnd) do
    local startX = direction == 1 and xStart or xEnd
    local endX = direction == 1 and xEnd or xStart

    local startPos = Vector3.new(startX, y, z)
    local endPos = Vector3.new(endX, y, z)

    tweenTo(startPos)
    tweenTo(endPos)

    z += zStep
    direction *= -1
end

task.wait(2)

-- Teleport player back to the lobby
local lobbyPlaceId = 116495829188952 -- PlaceId do lobby do Dead Rails [Alpha]
TeleportService:Teleport(lobbyPlaceId, player)
