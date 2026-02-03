local player = game.Players.LocalPlayer
local RunService = game:GetService("RunService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Workspace = game:GetService("Workspace")

local GoToBestHub = {}
GoToBestHub.Enabled = false
GoToBestHub.Connections = {}
GoToBestHub.isStopping = false
GoToBestHub.hasReachedHeight = false
GoToBestHub.targetHeight = nil
GoToBestHub.currentBestPet = nil
GoToBestHub.lastEquipTime = 0
GoToBestHub.lastSpamTime = 0
GoToBestHub.lastScanTime = 0

GoToBestHub.baseCoordinates = {
    ["Base 1"] = Vector3.new(-479.5, 13.4, -103.1),
    ["Base 2"] = Vector3.new(-339.8, 13.4, -98.5),
    ["Base 3"] = Vector3.new(-340.3, 13.4, 6.0),
    ["Base 4"] = Vector3.new(-340.0, 13.4, 113.1),
    ["Base 5"] = Vector3.new(-339.7, 14.0, 221.8),
    ["Base 6"] = Vector3.new(-479.3, 13.4, 218.5),
    ["Base 7"] = Vector3.new(-478.5, 13.4, 116.3),
    ["Base 8"] = Vector3.new(-478.2, 13.4, 5.8)
}

function GoToBestHub:extractValueGTB(text)
	if not text then return 0 end
	local clean = text:gsub("[%$,/s]", ""):gsub(",", "")
	local mult = 1
	if clean:match("[kK]") then
		mult = 1e3
		clean = clean:gsub("[kK]", "")
	elseif clean:match("[mM]") then
		mult = 1e6
		clean = clean:gsub("[mM]", "")
	elseif clean:match("[bB]") then
		mult = 1e9
		clean = clean:gsub("[bB]", "")
	elseif clean:match("[tT]") then
		mult = 1e12
		clean = clean:gsub("[tT]", "")
	end
	local num = tonumber(clean)
	return num and num * mult or 0
end

function GoToBestHub:findMostExpensiveBrainrot()
	if self.isStopping then return nil end
	
	local allPets = {}
	
	for _, obj in ipairs(Workspace:GetDescendants()) do
		if self.isStopping then break end
		
		if obj.Name == "AnimalOverhead" then
			local petValue = 0
			local petName = "Brainrot"
			
			for _, child in ipairs(obj:GetChildren()) do
				if child:IsA("TextLabel") then
					local text = child.Text
					if text and text:find("$") and text:find("/s") then
						petValue = self:extractValueGTB(text)
					end
					
					if child.Name == "DisplayName" and text ~= "" then
						petName = text
					end
				end
			end
			
			local position = nil
			local current = obj.Parent
			while current and current ~= Workspace do
				if current:IsA("BasePart") then
					position = current.Position
					break
				end
				current = current.Parent
			end
			
			if position and petValue > 0 then
				table.insert(allPets, {
					position = position,
					name = petName,
					value = petValue,
					baseName = self:findClosestBase(position)
				})
			end
		end
	end
	
	if #allPets > 0 and not self.isStopping then
		table.sort(allPets, function(a, b)
			return a.value > b.value
		end)
		
		local bestPet = allPets[1]
		return bestPet
	end
	
	return nil
end

function GoToBestHub:findClosestBase(petPosition)
	local closestBase = nil
	local closestDistance = math.huge
	
	for baseName, baseCoord in pairs(self.baseCoordinates) do
		local distance = (petPosition - baseCoord).Magnitude
		if distance < closestDistance then
			closestDistance = distance
			closestBase = baseName
		end
	end
	
	return closestBase
end

function GoToBestHub:equipGrappleHook()
    if self.isStopping then return nil end
    
    local now = tick()
    if now - self.lastEquipTime < 2 then return nil end
    
    local char = player.Character
    if not char then return nil end
    
    local backpack = player:FindFirstChild("Backpack")
    if not backpack then return nil end
    
    local grappleHook = backpack:FindFirstChild("Grapple Hook") or char:FindFirstChild("Grapple Hook")
    if not grappleHook then return nil end
    
    if grappleHook.Parent == backpack then
        local humanoid = char:FindFirstChildOfClass("Humanoid")
        if humanoid then
            humanoid:EquipTool(grappleHook)
            self.lastEquipTime = now
        end
    end
    
    return grappleHook
end

function GoToBestHub:spamGrappleHook()
    if self.isStopping then return end
    
    local now = tick()
    if now - self.lastSpamTime < 0.2 then return end
    
    local REUseItem = ReplicatedStorage:FindFirstChild("Packages")
    if REUseItem then
        REUseItem = REUseItem:FindFirstChild("Net")
        if REUseItem then
            REUseItem = REUseItem:FindFirstChild("RE/UseItem")
            if REUseItem then
                pcall(function()
                    REUseItem:FireServer(0.23450689315795897)
                    self.lastSpamTime = now
                end)
            end
        end
    end
end

function GoToBestHub:flyUpFirst()
    if self.isStopping then return end
    
    local char = player.Character
    if not char then return end
    
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    
    local startTime = tick()
    local startY = hrp.Position.Y
    local targetY = startY + 35
    
    while tick() - startTime < 3 and not self.isStopping do
        local currentY = hrp.Position.Y
        
        if currentY >= targetY then
            break
        end
        
        local upPos = Vector3.new(hrp.Position.X, targetY, hrp.Position.Z)
        local direction = (upPos - hrp.Position).Unit
        hrp.AssemblyLinearVelocity = direction * 40
        
        if tick() % 1 < 0.1 then
            self:spamGrappleHook()
        end
        
        task.wait(0.1)
    end
    
    self:stopMovement()
end

function GoToBestHub:flyToPosition(targetPos, speed)
    if self.isStopping then return 0 end
    local char = player.Character
    if not char then return 0 end
    
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return 0 end
    
    local direction = (targetPos - hrp.Position).Unit
    hrp.AssemblyLinearVelocity = direction * speed
    
    local distance = (hrp.Position - targetPos).Magnitude
    return distance
end

function GoToBestHub:equipFlyingCarpet()
    if self.isStopping then return nil end
    local char = player.Character
    if not char then return nil end
    
    local backpack = player:FindFirstChild("Backpack")
    if not backpack then return nil end
    
    local flyingCarpet = backpack:FindFirstChild("Flying Carpet")
    if not flyingCarpet then
        for _, item in ipairs(backpack:GetChildren()) do
            if (item.Name:lower():find("flying") or item.Name:lower():find("carpet")) and item:IsA("Tool") then
                flyingCarpet = item
                break
            end
        end
    end
    
    if flyingCarpet then
        local humanoid = char:FindFirstChildOfClass("Humanoid")
        if humanoid then
            humanoid:EquipTool(flyingCarpet)
            return flyingCarpet
        end
    end
    return nil
end

function GoToBestHub:teleportToBestBrainrot()
    if self.isStopping then return end
    
    local char = player.Character
    if not char then return end
    
    local humanoidRootPart = char:FindFirstChild("HumanoidRootPart")
    if not humanoidRootPart then return end
    
    if not self.currentBestPet then
        return
    end
    
    local flyingCarpet = self:equipFlyingCarpet()
    
    local targetBase = self.baseCoordinates[self.currentBestPet.baseName]
    if targetBase then
        humanoidRootPart.CFrame = CFrame.new(targetBase)
    else
        humanoidRootPart.CFrame = CFrame.new(self.currentBestPet.position)
    end
    
    task.wait(0.5)
    self:cleanStop()
end

function GoToBestHub:stopMovement()
    local char = player.Character
    if not char then return end
    
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    
    hrp.AssemblyLinearVelocity = Vector3.new(0, 0, 0)
end

function GoToBestHub:cleanStop()
    self.isStopping = true
    self.hasReachedHeight = false
    self.targetHeight = nil
    self.currentBestPet = nil
    self.lastEquipTime = 0
    self.lastSpamTime = 0
    self.lastScanTime = 0
    
    for name, conn in pairs(self.Connections) do
        if conn then
            conn:Disconnect()
            self.Connections[name] = nil
        end
    end
    
    self:stopMovement()
    task.wait(0.1)
    self.isStopping = false
    self.Enabled = false
end

function GoToBestHub:startSystem()
    self.isStopping = false
    self.hasReachedHeight = false
    self.targetHeight = nil
    self.lastEquipTime = 0
    self.lastSpamTime = 0
    self.lastScanTime = 0
    
    self.currentBestPet = self:findMostExpensiveBrainrot()
    
    if not self.currentBestPet then
        self:cleanStop()
        return
    end
    
    task.wait(0.5)
    
    self:equipGrappleHook()
    task.wait(0.5)
    
    self.Connections.movement = RunService.Heartbeat:Connect(function()
        if self.isStopping or not self.Enabled then return end
        
        local char = player.Character
        if not char then return end
        
        local hrp = char:FindFirstChild("HumanoidRootPart")
        if not hrp then return end
        
        local now = tick()
        
        if not self.hasReachedHeight then
            if not self.targetHeight then
                self.targetHeight = Vector3.new(hrp.Position.X, hrp.Position.Y + 35, hrp.Position.Z)
                self:flyUpFirst()
            end
            
            local distanceToHeight = (hrp.Position - self.targetHeight).Magnitude
            if distanceToHeight > 3 then
                self:flyToPosition(self.targetHeight, 50)
            else
                self.hasReachedHeight = true
                self:stopMovement()
                
                task.wait(0.1)
                self:teleportToBestBrainrot()
            end
        end
    end)
end

function GoToBestHub:Toggle()
    if self.isStopping then return end
    
    self.Enabled = not self.Enabled
    
    if self.Enabled then
        self:startSystem()
    else
        self:cleanStop()
    end
end

return GoToBestHub
