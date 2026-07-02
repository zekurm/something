local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")

-- 1. VISUAL SETTINGS CONFIGURATION
local Settings = {
	-- DIRECTIONAL RESTRAINTS DICTIONARIES MAPPED PER JOINT INDEX
	-- Note: All values are in degrees.
	-- Index 1 = RootJoint (Lower Spine) | Index 2 = TopToBottom (Upper Spine) | Index 3 = TopToHead (Head/Neck)
	Forwards = {
		[1] = {
			MaxLean = { Up = 2, Down = 10, Left = 0, Right = 0 }, -- Vertical pitch constraints (Up = sky arch, Down = floor bow)
			MaxTwist = { Up = 0, Down = 0, Left = 35, Right = 35 } -- Horizontal yaw constraints (Left/Right limits)
		},
		[2] = {
			MaxLean = { Up = 60, Down = 60, Left = 0, Right = 0 }, -- Increased head limits so it can track freely
			MaxTwist = { Up = 0, Down = 0, Left = 60, Right = 60 }
		},
	},
	Backwards = {
		[1] = {
			MaxLean = { Up = 10, Down = 25, Left = 0, Right = 0 },
			MaxTwist = { Up = 0, Down = 0, Left = 25, Right = 25 }
		},
		[2] = {
			MaxLean = { Up = 35, Down = 50, Left = 0, Right = 0 },
			MaxTwist = { Up = 0, Down = 0, Left = 35, Right = 35 }
		},
	},

	-- Posture & Mechanics
	BendBackwardsWhenBehind = true,
	BackwardBendAngle = 0,
	BackwardTwistMultiplier = 1.5,

	-- UNIFIED PER-JOINT DYNAMICS SETTINGS BLOCK
	LimbDynamics = {
		[1] = { 
			Speed = 90,  
			Smoothness = 0.8, 
			TweenDuration = 0.8, 
			EasingStyle = Enum.EasingStyle.Cubic, 
			EasingDirection = Enum.EasingDirection.Out 
		}, -- Lower Spine
		[2] = { 
			Speed = 500, 
			Smoothness = 1, 
			TweenDuration = .05, 
			EasingStyle = Enum.EasingStyle.Cubic, 
			EasingDirection = Enum.EasingDirection.Out 
		}  -- Neck / Head
	},

	-- Features & Offsets
	UseLineOfSight = true,      
	FaceDirectionOffset = Vector3.new(0, 0, 0) 
}

-- 2. FETCH PARTS FROM FOLDER
local folder = script.Parent
local target = folder:WaitForChild("target")
local rootPart = folder:WaitForChild("HumanoidRootPart")

local trackingTable = {
	folder:WaitForChild("Torso"),
	folder:WaitForChild("Head")
}

local motor6Ds = {
	rootPart:WaitForChild("RootJoint"),           
	trackingTable[1]:WaitForChild("Neck")   
}

-- Calculate total spine height for stable virtual targeting
local totalSpineLength = 0
for i, part in ipairs(trackingTable) do
	if i > 1 then
		totalSpineLength = totalSpineLength + (part.Position - trackingTable[i-1].Position).Magnitude
	else
		totalSpineLength = totalSpineLength + (part.Position - rootPart.Position).Magnitude
	end
end

-- Tables to house unique, independent frame channels safely per individual motor joint
local currentX = {0, 0, 0}
local currentY = {0, 0, 0}

local startX = {0, 0, 0}
local startY = {0, 0, 0}

local targetXData = {0, 0, 0}
local targetYData = {0, 0, 0}

-- Separated tracking elapsed timelines per individual bone index
local tweenTimeElapsed = {0, 0, 0}

-- Cache Raycast Parameters for optimization
local raycastParams = RaycastParams.new()
raycastParams.FilterType = Enum.RaycastFilterType.Exclude

-- Helper function to evaluate and apply asymmetrical directional clamps safely
local function clampAsymmetrical(angle, upDeg, downDeg, leftDeg, rightDeg, isTwist)
	if isTwist then
		local minLimit = -math.rad(rightDeg)
		local maxLimit = math.rad(leftDeg)
		return math.clamp(angle, minLimit, maxLimit)
	else
		local minLimit = -math.rad(downDeg)
		local maxLimit = math.rad(upDeg)
		return math.clamp(angle, minLimit, maxLimit)
	end
end

-- Helper function to clamp change per second (Angular Velocity Cap)
local function stepTowards(current, goal, maxChange)
	return current + math.clamp(goal - current, -maxChange, maxChange)
end

-- 3. MAIN UPDATE LOOP
RunService.Heartbeat:Connect(function(deltaTime)
	if not target or not rootPart or #trackingTable == 0 then return end

	local headPart = trackingTable[#trackingTable]
	local targetPos = target.Position
	local hasLineOfSight = true

	-- A. OPTIMIZED RAYCAST WALL DETECTION
	if Settings.UseLineOfSight then
		raycastParams.FilterDescendantsInstances = {folder}

		local origin = headPart.Position
		local direction = targetPos - origin
		local raycastResult = workspace:Raycast(origin, direction, raycastParams)

		if raycastResult and raycastResult.Instance ~= target then
			hasLineOfSight = false
		end
	end

	-- Predict a completely stable virtual head position above the RootPart
	local stableVirtualHeadPos = rootPart.Position + (rootPart.CFrame.UpVector * totalSpineLength)

	-- Get the vector pointing from the character to the ball, in local space
	local localTargetVector = rootPart.CFrame:VectorToObjectSpace(targetPos - stableVirtualHeadPos)

	-- 2D distances relative to the creature's facing orientation
	local forwardDistance = -localTargetVector.Z 
	local horizontalDistance = localTargetVector.X
	local totalHorizontalDist = math.sqrt(forwardDistance^2 + horizontalDistance^2)

	-- Compute behind-blend factor
	local rearCheckFactor = math.clamp((forwardDistance / math.max(totalHorizontalDist, 0.001)), -1, 1)
	local backwardBlendWeight = math.clamp((-rearCheckFactor + 1) / 2, 0, 1)

	-- Pre-declare baseline variables to safely prevent scoping math nil crashes
	local rawSpineX, rawSpineY = 0, 0
	local rawHeadX, rawHeadY = 0, 0

	-- Only calculate tracking angles if the creature can actively see the target
	if hasLineOfSight then
		-- LINEAR LEAN MATH (X Axis)
		local rawPitch = math.atan2(localTargetVector.Y, totalHorizontalDist)
		local targetLeanX = rawPitch

		-- TWIST MATH (Y Axis)
		local targetTwistY = math.atan2(-localTargetVector.X, -localTargetVector.Z)

		-- SEAMLESS POSTURE INTERPOLATION TRANSITION
		if Settings.BendBackwardsWhenBehind then
			local goalBackwardLean = -math.rad(Settings.BackwardBendAngle)
			targetLeanX = (rawPitch * (1 - backwardBlendWeight)) + (goalBackwardLean * backwardBlendWeight)
		end

		rawSpineX = targetLeanX
		rawSpineY = targetTwistY

		-- Calculate Head rotation values inside loop scope context
		local headLookAtWorld = CFrame.lookAt(headPart.Position, targetPos)
		local offsetRotation = CFrame.fromEulerAnglesXYZ(math.rad(Settings.FaceDirectionOffset.X), math.rad(Settings.FaceDirectionOffset.Y), math.rad(Settings.FaceDirectionOffset.Z))
		headLookAtWorld = headLookAtWorld * offsetRotation

		local parentPart = motor6Ds[#trackingTable].Part0
		local goalHeadLocalCFrame = parentPart.CFrame:ToObjectSpace(headLookAtWorld)

		local lookVector = goalHeadLocalCFrame.LookVector

		-- FIXED: Restored completely raw angles here to let your per-joint table check evaluate the clamps natively
		rawHeadX = math.atan2(lookVector.Y, math.sqrt(lookVector.X^2 + lookVector.Z^2))
		rawHeadY = math.atan2(-lookVector.X, -lookVector.Z)
	end

	-- Compute backward twist spine blending multiplier values once
	local baseFactor = (1 - backwardBlendWeight) * 0.6 + (backwardBlendWeight * Settings.BackwardTwistMultiplier)

	-- Decide which directional profile to use based on relative target positioning
	local trackingMode = (forwardDistance < 0 and Settings.BendBackwardsWhenBehind) and "Backwards" or "Forwards"
	local constraintProfile = Settings[trackingMode]

	-- B. INDEPENDENT PER-JOINT GOAL COMPILATION & CLAMPING
	local goalXData = {}
	local goalYData = {}

	for i = 1, #trackingTable do
		local jointLimits = constraintProfile[i]
		if jointLimits then
			if i == #trackingTable then
				-- Head tracking limits check (Now safely reads your custom sub-table limits)
				goalXData[i] = clampAsymmetrical(rawHeadX, jointLimits.MaxLean.Up, jointLimits.MaxLean.Down, jointLimits.MaxLean.Left, jointLimits.MaxLean.Right, false)
				goalYData[i] = clampAsymmetrical(rawHeadY, jointLimits.MaxTwist.Up, jointLimits.MaxTwist.Down, jointLimits.MaxTwist.Left, jointLimits.MaxTwist.Right, true)
			else
				-- Spine parts progressive distribution weight checks
				local distributedX = rawSpineX * (i / (#trackingTable - 1) * 0.6)
				local distributedY = rawSpineY * (i / (#trackingTable - 1) * baseFactor)

				goalXData[i] = clampAsymmetrical(distributedX, jointLimits.MaxLean.Up, jointLimits.MaxLean.Down, jointLimits.MaxLean.Left, jointLimits.MaxLean.Right, false)
				goalYData[i] = clampAsymmetrical(distributedY, jointLimits.MaxTwist.Up, jointLimits.MaxTwist.Down, jointLimits.MaxTwist.Left, jointLimits.MaxTwist.Right, true)
			end
		else
			goalXData[i] = 0
			goalYData[i] = 0
		end
	end

	-- C. INDEPENDENT TWEEN TIME SCHEDULING
	for i = 1, #trackingTable do
		if goalXData[i] ~= targetXData[i] or goalYData[i] ~= targetYData[i] then
			startX[i] = currentX[i]
			startY[i] = currentY[i]
			targetXData[i] = goalXData[i]
			targetYData[i] = goalYData[i]
			tweenTimeElapsed[i] = 0 
		end
	end

	-- D. SEPARATED SPEED CAP & TRANSFORM EVALUATION
	for i = 1, #trackingTable do
		local motor = motor6Ds[i]
		if motor and motor.Part0 and motor.Part1 then

			local config = Settings.LimbDynamics[i] or { 
				Speed = 180, 
				Smoothness = 0.5, 
				TweenDuration = 0.5, 
				EasingStyle = Enum.EasingStyle.Linear, 
				EasingDirection = Enum.EasingDirection.Out 
			}

			tweenTimeElapsed[i] = tweenTimeElapsed[i] + deltaTime
			local rawAlpha = math.clamp(tweenTimeElapsed[i] / math.max(config.TweenDuration, 0.001), 0, 1)
			local tweenAlpha = TweenService:GetValue(rawAlpha, config.EasingStyle, config.EasingDirection)

			local tweenTargetX = startX[i] + (targetXData[i] - startX[i]) * tweenAlpha
			local tweenTargetY = startY[i] + (targetYData[i] - startY[i]) * tweenAlpha

			local maxJointFrameChange = math.rad(config.Speed) * deltaTime
			currentX[i] = stepTowards(currentX[i], tweenTargetX, maxJointFrameChange)
			currentY[i] = stepTowards(currentY[i], tweenTargetY, maxJointFrameChange)
			currentX[i] = currentX[i] + (tweenTargetX - currentX[i]) * config.Smoothness
			currentY[i] = currentY[i] + (tweenTargetY - currentY[i]) * config.Smoothness
			motor.Transform = CFrame.fromEulerAnglesYXZ(currentX[i], currentY[i], 0)
		end
	end
end)
