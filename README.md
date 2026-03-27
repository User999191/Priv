local getNil = function(name, class) for _, v in next, getnilinstances() do if v.ClassName == class and v.Name == name then return v end end end

local Players = game:GetService("Players")
local player = Players.LocalPlayer
local RunService = game:GetService("RunService")

local RANGE = 200
local MAX_NPCS_PER_FRAME = 30

local function isNPC(model)
    if not (model and model:IsA("Model")) then return false end
    local hum = model:FindFirstChildOfClass("Humanoid")
    if not hum then return false end
    return Players:GetPlayerFromCharacter(model) == nil
end

local function getRoot(model)
    if not model then return nil end
    return model:FindFirstChild("HumanoidRootPart")
        or model:FindFirstChild("UpperTorso")
        or model:FindFirstChild("Torso")
end

local function getBestHitPart(npc)
    local priority = {"Head", "UpperTorso", "Torso", "HumanoidRootPart", "Right Arm"}
    for _, name in ipairs(priority) do
        local part = npc:FindFirstChild(name)
        if part and part:IsA("BasePart") then return part end
    end
    return npc:FindFirstChildWhichIsA("BasePart")
end

local function getAllNPCs()
    local npcs = {}
    local monstersFolder = workspace:FindFirstChild("Monsters")
    if monstersFolder then
        for _, obj in ipairs(monstersFolder:GetDescendants()) do
            if isNPC(obj) then table.insert(npcs, obj) end
        end
    end
    for _, folderName in ipairs({"Enemies", "Mobs", "Zombies"}) do
        local folder = workspace:FindFirstChild(folderName)
        if folder then
            for _, obj in ipairs(folder:GetDescendants()) do
                if isNPC(obj) then table.insert(npcs, obj) end
            end
        end
    end
    return npcs
end

local function getCurrentTool()
    local char = player.Character
    if not char then return nil, nil end
    for _, tool in ipairs(char:GetChildren()) do
        if tool:IsA("Tool") then
            local hitEvent = tool:FindFirstChild("HitEvent")
            if hitEvent then return tool, hitEvent end
            local server = tool:FindFirstChild("Server")
            if server then
                local inflict = server:FindFirstChild("InflictTarget")
                if inflict then return tool, inflict end
            end
        end
    end
    return nil, nil
end

local currentTool, currentRemote = nil, nil

local function updateTool()
    currentTool, currentRemote = getCurrentTool()
end

player.CharacterAdded:Connect(function()
    task.wait(0.5)
    updateTool()
end)

updateTool()

local npcList = {}
local index = 1
local lastCheck = tick()

-- Auto-apply to new NPCs
local descConn
local function startAutoDetect()
    if descConn then descConn:Disconnect() end
    descConn = workspace.DescendantAdded:Connect(function(obj)
        if obj:IsA("Humanoid") then
            local m = obj.Parent
            if isNPC(m) and not table.find(npcList, m) then
                table.insert(npcList, m)
            end
        elseif obj:IsA("BasePart") and (obj.Name == "HumanoidRootPart" or obj.Name == "UpperTorso" or obj.Name == "Torso") then
            local m = obj:FindFirstAncestorOfClass("Model")
            if isNPC(m) and not table.find(npcList, m) then
                table.insert(npcList, m)
            end
        end
    end)
end

startAutoDetect()

RunService.Heartbeat:Connect(function()
    local char = player.Character
    if not char or not char:FindFirstChild("Humanoid") or char.Humanoid.Health <= 0 then
        char = player.Character or player.CharacterAdded:Wait()
        updateTool()
        return
    end
    
    if not currentRemote then
        updateTool()
        return
    end
    
    local myRoot = char:FindFirstChild("HumanoidRootPart")
    if not myRoot then return end
    
    -- Update full NPC list every 1 second
    if tick() - lastCheck > 1 then
        npcList = getAllNPCs()
        lastCheck = tick()
        index = 1
    end
    
    if #npcList == 0 or index > #npcList then
        npcList = getAllNPCs()
        index = 1
    end
    
    local processed = 0
    while processed < MAX_NPCS_PER_FRAME and index <= #npcList do
        local npc = npcList[index]
        index = index + 1
        
        if npc and npc.Parent then
            local root = getRoot(npc)
            if not root then 
                root = npc:FindFirstChild("HumanoidRootPart") or npc:FindFirstChildWhichIsA("BasePart")
            end
            
            if root then
                local dist = (root.Position - myRoot.Position).Magnitude
                if dist <= RANGE and dist >= 2 then
                    local hitPart = getBestHitPart(npc)
                    local targetHum = npc:FindFirstChild("Humanoid")
                    
                    if hitPart and targetHum and targetHum.Health > 0 then
                        local direction = (hitPart.Position - myRoot.Position).Unit
                        direction = direction + Vector3.new(math.random(-8,8)/100, math.random(-5,5)/100, math.random(-8,8)/100)
                        direction = direction.Unit
                        
                        local args
                        if currentRemote.Name == "HitEvent" then
                            args = {npc, targetHum}
                        else
                            args = {hitPart, targetHum, direction}
                        end
                        
                        pcall(function()
                            currentRemote:FireServer(unpack(args))
                        end)
                    end
                end
            end
        end
        processed = processed + 1
    end
end)
