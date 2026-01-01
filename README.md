local Workspace = game:GetService("Workspace")
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")
local GuiService = game:GetService("GuiService")
local TweenService = game:GetService("TweenService")

local camera = Workspace.CurrentCamera

local localPlayer = assert(Players.LocalPlayer, "LocalPlayer is nil.")
local playerGui = localPlayer:WaitForChild("PlayerGui")

local rootScreenGui = playerGui:WaitForChild("RootScreenGui")
local mobileCombatContainer = rootScreenGui:WaitForChild("MobileCombatContainer")
local canvasGroup = mobileCombatContainer:WaitForChild("CanvasGroup")
local runButton = canvasGroup:WaitForChild("RunButton")
local dashButton = canvasGroup:WaitForChild("DashButton")
local swingButton = canvasGroup:WaitForChild("SwingButton")
local uptiltButton = canvasGroup:WaitForChild("UptiltButton")
local blockButton = canvasGroup:WaitForChild("BlockButton")

local config = ReplicatedStorage.Config

local constants = config.Constants
local ClientConstants = require(constants.ClientConstants)
local SharedConstants = require(constants.SharedConstants)

local templates = config.Templates
local SharedTemplates = require(templates.SharedTemplates)

local sharedActions = config.SharedActions
local PlayerDataMapActions = require(sharedActions.PlayerDataMap)
local PlayerEphemeralDataMapActions = require(sharedActions.PlayerEphemeralDataMap)

local utils = ReplicatedStorage.Utils
local Packets = require(utils.Packets)
local ClientHelperFunctions = require(utils.ClientHelperFunctions)
local SharedHelperFunctions = require(utils.SharedHelperFunctions)
local SharedConfigFunctions = require(utils.SharedConfigFunctions)
local SharedStateFunctions = require(utils.SharedStateFunctions)
local AnimationFunctions = require(utils.AnimationFunctions)
local AnimationHandler = require(utils.AnimationHandler)
local SfxFunctions = require(utils.SfxFunctions)
local VfxFunctions = require(utils.VfxFunctions)
local ViewportFunctions = require(utils.ViewportFunctions)
local CameraFunctions = require(utils.CameraFunctions)

local classes = ReplicatedStorage.Classes
local RadialMenuClass = require(classes.RadialMenu)

local packages = ReplicatedStorage.Packages
local Sift = require(packages.Sift)

local typeExports = ReplicatedStorage.TypeExports
local ClientTypes = require(typeExports.ClientTypes)
local SharedTypes = require(typeExports.SharedTypes)

type Packet = Packets.Packet

type InputType = ClientTypes.InputType

type CurrencyType = SharedTypes.CurrencyType
type PlayerFlagType = SharedTypes.PlayerFlagType
type PlayerSettingType = SharedTypes.PlayerSettingType
type CodeType = SharedTypes.CodeType
type OmnitrixType = SharedTypes.OmnitrixType
type AlienType = SharedTypes.AlienType
type AlienSkinType = SharedTypes.AlienSkinType
type AlienStatType = SharedTypes.AlienStatType
type AlienAbilityType = SharedTypes.AlienAbilityType

type PlayerState = SharedTypes.PlayerState
type MovementState = SharedTypes.MovementState
type CombatState = SharedTypes.CombatState

type PlayerDataMap = SharedTypes.PlayerDataMap
type PlayerData = SharedTypes.PlayerData
type PlayerSaveSlotData = SharedTypes.PlayerSaveSlotData
type PlayerSaveSlotEntryData = SharedTypes.PlayerSaveSlotEntryData
type PlayerStatsData = SharedTypes.PlayerStatsData
type PlayerCurrencyData = SharedTypes.PlayerCurrencyData
type PlayerFlagsData = SharedTypes.PlayerFlagsData
type PlayerSettingsData = SharedTypes.PlayerSettingsData
type PlayerHistoryData = SharedTypes.PlayerHistoryData
type PlayerCodeHistoryData = SharedTypes.PlayerCodeHistoryData
type PlayerSpecData = SharedTypes.PlayerSpecData
type PlayerOmnitrixDataMap = SharedTypes.PlayerOmnitrixDataMap
type PlayerAlienDataMap = SharedTypes.PlayerAlienDataMap
type PlayerOmnitrixData = SharedTypes.PlayerOmnitrixData
type PlayerAlienData = SharedTypes.PlayerAlienData
type PlayerGamePassHistoryData = SharedTypes.PlayerGamePassHistoryData
type GamePassPurchaseData = SharedTypes.GamePassPurchaseData
type PlayerDevProductHistoryData = SharedTypes.PlayerDevProductHistoryData
type DevProductPurchaseData = SharedTypes.DevProductPurchaseData

type PlayerEphemeralDataMap = SharedTypes.PlayerEphemeralDataMap
type PlayerEphemeralData = SharedTypes.PlayerEphemeralData

type OmnitrixConfig = SharedTypes.OmnitrixConfig
type OmnitrixAlienPoolConfig = SharedTypes.OmnitrixAlienPoolConfig
type OmnitrixAlienPoolItem = SharedTypes.OmnitrixAlienPoolItem
type OmnitrixViewportRotationConfig = SharedTypes.OmnitrixViewportRotationConfig
type AlienConfig = SharedTypes.AlienConfig
type AlienSkinMapConfig = SharedTypes.AlienSkinMapConfig
type AlienSkinConfig = SharedTypes.AlienSkinConfig
type AlienStatConfig = SharedTypes.AlienStatConfig
type AlienStatOverrideConfig = SharedTypes.AlienStatOverrideConfig
type AlienAbilityMapConfig = SharedTypes.AlienAbilityMapConfig
type AlienAbilityConfig = SharedTypes.AlienAbilityConfig

local PREFERRED_INPUT = ClientConstants.PREFERRED_INPUT
local GAMEPAD_INPUT = ClientConstants.GAMEPAD_INPUT
local MOUSE_INPUT = ClientConstants.MOUSE_INPUT
local KEYBOARD_INPUT = ClientConstants.KEYBOARD_INPUT
local TOUCH_INPUT = ClientConstants.TOUCH_INPUT

local SPRINT_KEY_CODE = ClientConstants.SPRINT_KEY_CODE
local DASH_KEY_CODE = ClientConstants.DASH_KEY_CODE
local BLOCK_KEY_CODE = ClientConstants.BLOCK_KEY_CODE
local UPTILT_KEY_CODE = ClientConstants.UPTILT_KEY_CODE

local playerEphemeralDataTemplate = SharedTemplates.playerEphemeralData

local preferredCombatContainer = nil

local combatContainers = {
	Gamepad = nil,
	MouseKeyboard = nil,
	Touch = mobileCombatContainer,
}

local movementAnimationNames = {
	"Sprint",
	"Dash/W",
	"Dash/A",
	"Dash/S",
	"Dash/D",
}

local dashKeyCodes = {
	Enum.KeyCode.W,
	Enum.KeyCode.A,
	Enum.KeyCode.S,
	Enum.KeyCode.D,
}

local dashKeyCodesToAnimationNames = {
	[Enum.KeyCode.W] = "Dash/W",
	[Enum.KeyCode.A] = "Dash/A",
	[Enum.KeyCode.S] = "Dash/S",
	[Enum.KeyCode.D] = "Dash/D",
}

local dashKeyCodesToCFrameOffsets = {
	[Enum.KeyCode.W] = CFrame.new(0, 0, -50),
	[Enum.KeyCode.A] = CFrame.new(-50, 0, 0),
	[Enum.KeyCode.S] = CFrame.new(0, 0, 50),
	[Enum.KeyCode.D] = CFrame.new(50, 0, 0),
}

local dashTweenInfo = TweenInfo.new(.3, Enum.EasingStyle.Sine, Enum.EasingDirection.In, 0, false, 0)

local Interface = {}

function Interface.init()
	Interface.onPlayerEphemeralDataMapChanged(PlayerEphemeralDataMapActions.getPlayerEphemeralDataMap(), {})
end

function Interface.show()
	if not preferredCombatContainer then return end

	preferredCombatContainer.Visible = true
	ClientHelperFunctions.fadeInCanvasGroup(preferredCombatContainer)
end

function Interface.hide()
	if not preferredCombatContainer then return end

	ClientHelperFunctions.fadeOutCanvasGroup(preferredCombatContainer)
	preferredCombatContainer.Visible = false
end

function Interface.isVisible()
	return preferredCombatContainer and preferredCombatContainer.Visible
end

function Interface.updatePreferredCombatContainer(targetInputType)
	local combatContainer = combatContainers[targetInputType]
	if not combatContainer then return end

	preferredCombatContainer = combatContainer
end

function Interface.stopAllMovementAnimations(player)
	if localPlayer ~= player then return end

	local character = player.Character
	if not character then return end

	for _, animationName in movementAnimationNames do
		AnimationFunctions.stopAnimationTrack(character, "Movement/" .. animationName)
	end
end

function Interface.onPlayerEphemeralDataInitialized(player, playerEphemeralData)

end

function Interface.onPlayerEphemeralDataChanged(player, newPlayerEphemeralData, oldPlayerEphemeralData)

end

function Interface.onPlayerEphemeralDataMapChanged(newPlayerEphemeralDataMap, oldPlayerEphemeralDataMap)
	for userId, newPlayerEphemeralData in newPlayerEphemeralDataMap do
		local player = Players:GetPlayerByUserId(tonumber(userId))
		if not player then continue end

		local oldPlayerEphemeralData = oldPlayerEphemeralDataMap[userId]
		if oldPlayerEphemeralData then
			Interface.onPlayerEphemeralDataChanged(player, newPlayerEphemeralData, oldPlayerEphemeralData)
			continue
		end

		Interface.onPlayerEphemeralDataInitialized(player, newPlayerEphemeralData)
	end
end

function Interface.getCurrentAlienType()
	local ephemeralData = SharedStateFunctions.getPlayerEphemeralData(localPlayer)
	return ephemeralData and ephemeralData.selectedAlienType
end

function Interface.requestDash()
	local character = localPlayer.Character
	if not character then return end

	local currentAlien = Interface.getCurrentAlienType()

	for dashKeyCode, dashAnimationName in dashKeyCodesToAnimationNames do
		if not KEYBOARD_INPUT:IsKeyDown(dashKeyCode) then continue end

		local dashCFrameOffset = dashKeyCodesToCFrameOffsets[dashKeyCode]
		local dashTween = TweenService:Create(character.PrimaryPart, dashTweenInfo, { CFrame = character:GetPivot() * dashCFrameOffset })

		dashTween:Play()

		if currentAlien then
			AnimationFunctions.playAlienAnimation(character, "Dash", currentAlien)
		else
		local success = pcall(function()
			AnimationFunctions.playAnimationTrack(character, "Movement/Dash/" .. dashAnimationName)
		end)
		if not success then
			warn(`Failed to play dash animation: {dashAnimationName}`)
		end
		end

		break
	end

	Packets.RequestToDash:Fire(Enum.KeyCode.W)
end

function Interface.requestM1()
	local character = localPlayer.Character
	if not character then return end

	local currentAlien = Interface.getCurrentAlienType()

	Packets.RequestToM1:Fire()

	if currentAlien then
		AnimationFunctions.playRandomPunch(character, currentAlien)
	else
		AnimationFunctions.playRandomPunch(character, nil)
	end
end

function Interface.requestUptilt()
	local character = localPlayer.Character
	if not character then return end

	local currentAlien = Interface.getCurrentAlienType()

	Packets.RequestToUptilt:Fire()

	if currentAlien then
		AnimationFunctions.playAlienAnimation(character, "Uptilt", currentAlien)
	else
		local success = pcall(function()
			AnimationFunctions.playAnimationTrack(character, "Combat/Uptilt")
		end)
		if not success then
			warn("Failed to play Uptilt animation")
		end
	end
end

function Interface.requestBlock()
	local character = localPlayer.Character
	if not character then return end

	local currentAlien = Interface.getCurrentAlienType()

	Packets.RequestToBlock:Fire()

	if currentAlien then
		AnimationFunctions.playAlienAnimation(character, "Block", currentAlien)
	else
		local success = pcall(function()
			AnimationFunctions.playAnimationTrack(character, "Combat/Block")
		end)
		if not success then
			warn("Failed to play Block animation")
		end
	end
end

function Interface.registerEventListeners()
	PREFERRED_INPUT.Observe(Interface.updatePreferredCombatContainer)

	MOUSE_INPUT.LeftDown:Connect(function()
		if KEYBOARD_INPUT:IsKeyDown(UPTILT_KEY_CODE) then
			Interface.requestUptilt()
		else
			Interface.requestM1()
		end
	end)

	KEYBOARD_INPUT.KeyDown:Connect(function(keyCode)
		if keyCode == SPRINT_KEY_CODE then
			Packets.RequestToStartSprinting:Fire()

			while KEYBOARD_INPUT:IsKeyDown(SPRINT_KEY_CODE) do
				task.wait()
			end

			Packets.RequestToStopSprinting:Fire()
		elseif keyCode == DASH_KEY_CODE then
			Interface.requestDash()
		elseif keyCode == BLOCK_KEY_CODE then
			Packets.RequestToBlock:Fire()
		end
	end)

	PlayerEphemeralDataMapActions.onPlayerEphemeralDataMapChanged(Interface.onPlayerEphemeralDataMapChanged)

	ClientHelperFunctions.onGuiObjectPressed(runButton, function()
		Packets.RequestToStartSprinting:Fire()
	end)

	ClientHelperFunctions.onGuiObjectReleased(runButton, function()
		Packets.RequestToStopSprinting:Fire()
	end)

	ClientHelperFunctions.onGuiObjectPressed(dashButton, function()
		Interface.requestDash()
	end)

	ClientHelperFunctions.onGuiObjectPressed(swingButton, function()
		Interface.requestM1()
	end)

	ClientHelperFunctions.onGuiObjectPressed(uptiltButton, function()
		Interface.requestUptilt()
	end)

	ClientHelperFunctions.onGuiObjectPressed(blockButton, function()
		Interface.requestBlock()
	end)

	AnimationHandler.init()
end

return Interface
