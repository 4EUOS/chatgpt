StarterGUI:

UI CLIENT:

-- ================================================
-- UICLIENT (StarterGui/UI - LocalScript) FINAL v3.0
-- ================================================
-- Master client controller for Manager Mode.
-- Refactored for clean Remotes, AAA animations, 
-- and World Map Zoom Tutorial.
-- ================================================
local Players         = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Player          = Players.LocalPlayer

local UI             = script.Parent
local Core           = UI:WaitForChild("Core")
local PagesFolder    = UI:WaitForChild("Pages")

-- Core Controllers
local PageManager    = require(Core:WaitForChild("PageManager"))
local UIController   = require(Core:WaitForChild("UIController"))
local TweenWrapper   = require(Core:WaitForChild("TweenServiceWrapper"))
local Tutorial       = require(Core:WaitForChild("TutorialController"))
local AdminPanel     = require(Core:WaitForChild("AdminPanel")) -- [NEW]

-- ================================================
-- 🛡️ REMOTES (Clean Folder Structure)
-- ================================================
local Remotes         = ReplicatedStorage:WaitForChild("Remotes")
local RequestData    = Remotes:WaitForChild("RequestData")
local UpdateData     = Remotes:WaitForChild("UpdateData")
local UIEvent        = Remotes:WaitForChild("UIEvent")

-- Legacy function aliases (pointing to new Remotes folder)
local function getRemote(name) return Remotes:WaitForChild(name, 8) end

local StatUpdateEvent   = getRemote("StatUpdate")
local TrainingRequest   = getRemote("TrainingRequest")
local StartMatch        = getRemote("StartMatch")
local SimulateMatch     = getRemote("SimulateMatch")
local MatchUpdate       = getRemote("MatchUpdate")
local SetCareerMode     = getRemote("SetCareerMode")
local GetLeagueData     = getRemote("GetLeagueData")
local GetTransferMarket = getRemote("GetTransferMarket")
local GetAcademyData    = getRemote("GetAcademyData")
local PromoteYouth      = getRemote("PromoteYouth")
local ReleaseYouth      = getRemote("ReleaseYouth")
local UpgradeFacility   = getRemote("UpgradeFacility")
local SendScout         = getRemote("SendScout")
local GetSponsorData    = getRemote("GetSponsorData")
local ClaimSponsor      = getRemote("ClaimSponsor")
local GetWorldNews      = getRemote("GetWorldNews")
local SetTactics        = getRemote("SetTactics")
local GetNextMatch      = getRemote("GetNextMatch")

-- ================================================
-- 📦 CLIENT STATE
-- ================================================
local ClubProfile            = nil  -- Master Data Object
local CurrentMode            = "MANAGER"
local MatchIsLive            = false
local SelectedTransferPlayer = nil
local SelectedTacticsPlayer  = nil
local CurrentScoutResults    = {}

-- Tab cycle state
local TABS = {"Overview","Squad","Scouting","Academy","Finances","Market","Sponsors","Tactics","World"}
local CurrentTab = "Overview"

-- ================================================
-- 🎯 PAGE REGISTRATION
-- ================================================
for _, f in ipairs(PagesFolder:GetChildren()) do
	if f:IsA("Frame") or f:IsA("CanvasGroup") then
		PageManager:RegisterPage(f.Name, f)
	end
end

-- Page refs
local MainMenu      = PagesFolder:WaitForChild("MainMenu")
local ManagerDash   = PagesFolder:WaitForChild("ManagerDashboard")
local ClubCreator   = PagesFolder:FindFirstChild("ClubCreator")
local MatchSimPage  = PagesFolder:WaitForChild("MatchSim")
local SettingsPage  = PagesFolder:FindFirstChild("Settings")

-- Manager Dashboard inner refs
local TabBar        = ManagerDash:FindFirstChild("TabBar")
local ContentArea   = ManagerDash:FindFirstChild("ContentArea")
local TabSlider     = TabBar and TabBar:FindFirstChild("Slider")
local TopBar        = ManagerDash:FindFirstChild("TopBar")

-- Match refs
local HUD           = MatchSimPage:FindFirstChild("HUD")
local KickOffBtn    = MatchSimPage:FindFirstChild("KickOffButton")
local SimBtn        = MatchSimPage:FindFirstChild("SimulateButton")

-- ================================================
-- 🎨 UTILITY
-- ================================================
local NEON      = Color3.fromRGB(0, 255, 154)
local RED       = Color3.fromRGB(255, 60, 80)
local DARK      = Color3.fromRGB(12, 12, 18)
local SURFACE   = Color3.fromRGB(22, 22, 32)
local SURFACE2  = Color3.fromRGB(30, 30, 44)
local TEXT_DIM  = Color3.fromRGB(140, 150, 170)
local TEXT_ON   = Color3.new(1,1,1)

-- ================================================
-- 🎨 UTILITY (Themed per your sleek black & green)
-- ================================================
local NEON      = Color3.fromRGB(0, 255, 150)
local RED       = Color3.fromRGB(255, 60, 80)
local DARK      = Color3.fromRGB(15, 15, 20)
local SURFACE   = Color3.fromRGB(25, 25, 35)
local TEXT_DIM  = Color3.fromRGB(140, 150, 170)
local function tween(obj, props, t, style, dir)
	game:GetService("TweenService"):Create(obj, TweenInfo.new(t or 0.3, style or Enum.EasingStyle.Quad, dir or Enum.EasingDirection.Out), props):Play()
end

-- ================================================
-- 🧹 THE PURGE (Fix Overlaps)
-- ================================================
local function purgeOldUI(panel)
	if not panel then return end
	for _, child in ipairs(panel:GetChildren()) do
		if not child:IsA("UIBase") and not child.Name:match("Card") and not child.Name:match("Header") and not child.Name:match("Content") then
			child.Visible = false
		end
	end
end

local function getOrCreateHeader(parent, title)
	local h = parent:FindFirstChild("Header")
	if not h then
		h = Instance.new("TextLabel", parent)
		h.Name = "Header"
		h.Size = UDim2.new(1, -20, 0, 40)
		h.Position = UDim2.new(0, 10, 0, 10)
		h.BackgroundTransparency = 1
		h.Font = Enum.Font.Oswald
		h.TextColor3 = NEON
		h.TextSize = 24
		h.TextXAlignment = Enum.TextXAlignment.Left
	end
	h.Text = title:upper()
end

-- ================================================
-- 💎 AAA DESIGN SYSTEM (Glassmorphism)
-- ================================================
local function applyGlassStyle(frame)
	frame.BackgroundColor3 = Color3.fromRGB(20, 20, 30)
	frame.BackgroundTransparency = 0.3
	frame.BorderSizePixel = 0

	local corner = Instance.new("UICorner", frame)
	corner.CornerRadius = UDim.new(0, 12)

	local stroke = Instance.new("UIStroke", frame)
	stroke.Color = NEON
	stroke.Thickness = 1.5
	stroke.Transparency = 0.7

	-- Sub-glow
	local glow = Instance.new("UIStroke", frame)
	glow.Color = NEON
	glow.Thickness = 3
	glow.Transparency = 0.95
end

local function getOrCreateCard(parent, name, size, pos)
	local card = parent:FindFirstChild(name)
	if not card then
		card = Instance.new("Frame", parent)
		card.Name = name
		card.Size = size
		card.Position = pos
		applyGlassStyle(card)

		local TitleL = Instance.new("TextLabel", card)
		TitleL.Name = "HeaderTitle"
		TitleL.Size = UDim2.new(1, -20, 0, 30)
		TitleL.Position = UDim2.new(0, 12, 0, 5)
		TitleL.BackgroundTransparency = 1
		TitleL.Text = name:gsub("Card", ""):upper()
		TitleL.TextColor3 = NEON
		TitleL.Font = Enum.Font.Oswald
		TitleL.TextSize = 16
		TitleL.TextXAlignment = Enum.TextXAlignment.Left
	end
	return card
end

local function setupViewport(parent)
	local vp = parent:FindFirstChild("PlayerView")
	if not vp then
		vp = Instance.new("ViewportFrame", parent)
		vp.Name = "PlayerView"
		vp.Size = UDim2.new(1, 0, 0.8, 0)
		vp.Position = UDim2.new(0, 0, 0.2, 0)
		vp.BackgroundTransparency = 1
		vp.Ambient = Color3.new(1, 1, 1)
		vp.LightColor = NEON

		local cam = Instance.new("Camera", vp)
		vp.CurrentCamera = cam
		cam.CFrame = CFrame.new(Vector3.new(0, 5, 8), Vector3.new(0, 3, 0))
	end
	return vp
end

local function fmtMoney(v)
	if v >= 1000000 then return string.format("£%.2fM", v/1000000)
	elseif v >= 1000 then return string.format("£%.0fK", v/1000)
	else return "£" .. math.floor(v) end
end

local function moraleColor(m)
	if m >= 75 then return NEON
	elseif m >= 45 then return Color3.fromRGB(255,200,0)
	else return RED end
end

-- ================================================
-- 📑 TAB SYSTEM (AAA Transitions)
-- ================================================
-- [switchTab moved down]

if TabBar then
	for _, btn in ipairs(TabBar:GetChildren()) do
		if btn:IsA("TextButton") and btn.Name:sub(1,4) == "Tab_" then
			btn.MouseButton1Click:Connect(function()
				switchTab(btn.Name:sub(5))
			end)
		end
	end
end

-- ================================================
-- 🔄 DATA FETCHING
-- ================================================
local function fetchClubData()
	local ok, data = pcall(function() return RequestData:InvokeServer() end)
	if ok and data then ClubProfile = data end
	return ClubProfile
end

-- ================================================
-- 📊 OVERVIEW TAB (Smoothed & Optimized)
-- ================================================
local ListPool = {} -- [type] = { Model }

local function getFromPool(pool, parent)
	local item = pool[1]
	if item then
		table.remove(pool, 1)
		item.Visible = true
		item.Parent = parent
		return item
	end
	return nil
end

function refreshOverview()
	local data = fetchClubData()
	if not data then return end

	local panel = ContentArea:FindFirstChild("Panel_Overview")
	if not panel then return end
	purgeOldUI(panel)
	-- 1. Status Cards (Top Row)
	local budgetCard = getOrCreateCard(panel, "BudgetCard", UDim2.new(0.3, -15, 0, 100), UDim2.new(0, 10, 0, 60))
	local confCard   = getOrCreateCard(panel, "ConfidenceCard", UDim2.new(0.3, -15, 0, 100), UDim2.new(0.3, 5, 0, 60))
	local formCard   = getOrCreateCard(panel, "FormCard", UDim2.new(0.4, -20, 0, 100), UDim2.new(0.6, 10, 0, 60))

	local bVal = budgetCard:FindFirstChild("Value") or Instance.new("TextLabel", budgetCard)
	bVal.Name = "Value" bVal.Size = UDim2.new(1,0,0,40) bVal.Position = UDim2.new(0,0,0,40)
	bVal.BackgroundTransparency = 1 bVal.Font = Enum.Font.Oswald bVal.TextSize = 28 bVal.TextColor3 = Color3.new(1,1,1)
	TweenWrapper:CountUp(bVal, 0, data.Budget, 1.0, "£")

	-- Form Dots
	local fHolder = formCard:FindFirstChild("Dots") or Instance.new("Frame", formCard)
	fHolder.Name = "Dots" fHolder.Size = UDim2.new(1,-20,0,30) fHolder.Position = UDim2.new(0,10,0,50)
	fHolder.BackgroundTransparency = 1
	local fl = fHolder:FindFirstChildOfClass("UIListLayout") or Instance.new("UIListLayout", fHolder)
	fl.FillDirection = Enum.FillDirection.Horizontal fl.Padding = UDim.new(0,8) fl.HorizontalAlignment = Enum.HorizontalAlignment.Center

	for i, res in ipairs(data.RecentResults or {}) do
		if i > 5 then break end
		local d = fHolder:FindFirstChild("Dot_" .. i) or Instance.new("Frame", fHolder)
		d.Name = "Dot_" .. i d.Size = UDim2.new(0,25,0,25) Instance.new("UICorner", d).CornerRadius = UDim.new(1,0)
		d.BackgroundColor3 = (res == "W") and NEON or (res == "L" and RED or Color3.fromRGB(150,150,150))
		local t = d:FindFirstChild("Label") or Instance.new("TextLabel", d)
		t.Name = "Label" t.Size = UDim2.new(1,0,1,0) t.BackgroundTransparency = 1 t.Text = res t.TextColor3 = DARK t.Font = Enum.Font.GothamBold
	end

	-- 2. League Standing Card (Snippet)
	local leagueCard = getOrCreateCard(panel, "StandingCard", UDim2.new(0.66, -15, 1, -175), UDim2.new(0, 10, 0, 165))
	local sList = leagueCard:FindFirstChild("List") or Instance.new("ScrollingFrame", leagueCard)
	sList.Name = "List" sList.Size = UDim2.new(1,-24,1,-50) sList.Position = UDim2.new(0,12,0,40)
	sList.BackgroundTransparency = 1 sList.ScrollBarThickness = 1
	local lLay = sList:FindFirstChildOfClass("UIListLayout") or Instance.new("UIListLayout", sList)
	lLay.Padding = UDim.new(0, 4)

	for i, club in ipairs(data.Standings or {}) do
		if i > 8 then break end
		local r = Instance.new("Frame", sList)
		r.Size = UDim2.new(1,0,0,30) r.BackgroundTransparency = 0.8 r.BackgroundColor3 = (club.Name == data.ClubName) and NEON or SURFACE2
		Instance.new("UICorner", r).CornerRadius = UDim.new(0,6)
		local l = Instance.new("TextLabel", r)
		l.Size = UDim2.new(1,-20,1,0) l.Position = UDim2.new(0,10,0,0) l.BackgroundTransparency = 1 l.TextXAlignment = Enum.TextXAlignment.Left
		l.Font = Enum.Font.GothamMedium l.TextSize = 12 l.TextColor3 = (club.Name == data.ClubName) and DARK or Color3.new(1,1,1)
		l.Text = i .. ". " .. club.Name .. "  •  " .. (club.LeagueStats and club.LeagueStats.Points or 0) .. " PTS"
	end

	-- 3. Next Match Widget
	local nextCard = getOrCreateCard(panel, "NextMatchWidget", UDim2.new(0.34, -15, 1, -175), UDim2.new(0.66, 5, 0, 165))
	local ok, nextMatch = pcall(function() return GetNextMatch:InvokeServer() end)
	if ok and nextMatch then
		local oppL = nextCard:FindFirstChild("Opponent") or Instance.new("TextLabel", nextCard)
		oppL.Name = "Opponent" oppL.Size = UDim2.new(1,0,0,60) oppL.Position = UDim2.new(0,0,0.2,0)
		oppL.BackgroundTransparency = 1 oppL.Font = Enum.Font.Oswald oppL.TextSize = 24
		oppL.TextColor3 = Color3.new(1,1,1) oppL.TextXAlignment = Enum.TextXAlignment.Center 
		oppL.Text = "NEXT: " .. nextMatch.OpponentName:upper()

		local playBtn = nextCard:FindFirstChild("DashboardPlayBtn") or Instance.new("TextButton", nextCard)
		playBtn.Name = "DashboardPlayBtn" playBtn.Size = UDim2.new(0.8,0,0,45) playBtn.Position = UDim2.new(0.1,0,0.7,0)
		playBtn.BackgroundColor3 = NEON playBtn.TextColor3 = DARK playBtn.Font = Enum.Font.GothamBold
		playBtn.Text = "⚽ ENTER MATCH SIM" Instance.new("UICorner", playBtn)
		playBtn.MouseButton1Click:Connect(function() PageManager:SwitchPage("MatchSim", UIController) end)
	else
		local oppL = nextCard:FindFirstChild("Opponent") or Instance.new("TextLabel", nextCard)
		oppL.Name = "Opponent" oppL.Size = UDim2.new(1,0,1,0) oppL.Position = UDim2.new(0,0,0,0)
		oppL.BackgroundTransparency = 1 oppL.Font = Enum.Font.GothamMedium oppL.TextSize = 14
		oppL.TextColor3 = TEXT_DIM oppL.TextXAlignment = Enum.TextXAlignment.Center 
		oppL.Text = "OFF-WEEK: NO FIXTURE SCHEDULED"
	end
end

-- ================================================
-- 👥 SQUAD TAB (Pooled & Smooth)
-- ================================================
function refreshSquad()
	local data = fetchClubData()
	if not data or not data.Roster then return end

	local panel = ContentArea:FindFirstChild("Panel_Squad")
	if not panel then return end
	purgeOldUI(panel)
	getOrCreateHeader(panel, "Squad")

	-- Ensure Cards (Shifted Down to Y=60)
	local listCard    = getOrCreateCard(panel, "SquadListCard", UDim2.new(0.5, -20, 1, -70), UDim2.new(0, 10, 0, 60))
	local previewCard = getOrCreateCard(panel, "PreviewCard", UDim2.new(0.5, -10, 1, -70), UDim2.new(0.5, 5, 0, 60))

	-- Category Filters (NEW)
	local filterContainer = listCard:FindFirstChild("Filters") or Instance.new("Frame", listCard)
	filterContainer.Name = "Filters"
	filterContainer.Size = UDim2.new(1, -20, 0, 30)
	filterContainer.Position = UDim2.new(0, 10, 0, 35)
	filterContainer.BackgroundTransparency = 1
	local l = filterContainer:FindFirstChildOfClass("UIListLayout") or Instance.new("UIListLayout", filterContainer)
	l.FillDirection = Enum.FillDirection.Horizontal l.Padding = UDim.new(0, 5)

	local playerList = listCard:FindFirstChild("PlayerList")
	if not playerList then
		playerList = Instance.new("ScrollingFrame", listCard)
		playerList.Name = "PlayerList"
		playerList.Size = UDim2.new(1, -20, 1, -85)
		playerList.Position = UDim2.new(0, 10, 0, 75)
		playerList.BackgroundTransparency = 1
		playerList.CanvasSize = UDim2.new(0,0,0,1000)
		playerList.ScrollBarThickness = 2
		local ll = Instance.new("UIListLayout", playerList)
		ll.Padding = UDim.new(0, 8)
	end

	-- Return squad frames to pool
	if not ListPool["SquadRow"] then ListPool["SquadRow"] = {} end
	for _, child in ipairs(playerList:GetChildren()) do
		if child:IsA("Frame") then
			child.Visible = false
			table.insert(ListPool["SquadRow"], child)
		end
	end

	local sorted = {}
	for _, p in pairs(data.Roster) do table.insert(sorted, p) end
	table.sort(sorted, function(a,b) return a.OVR > b.OVR end)

	for i, player in ipairs(sorted) do
		local row = getFromPool(ListPool["SquadRow"], playerList)
		if not row then
			row = Instance.new("Frame")
			row.Size = UDim2.new(1, -8, 0, 50)
			row.BackgroundColor3 = SURFACE
			row.BackgroundTransparency = 0.5
			Instance.new("UICorner", row).CornerRadius = UDim.new(0, 8)

			local nL = Instance.new("TextLabel", row)
			nL.Name = "NameLabel"
			nL.Size = UDim2.new(0.6, 0, 0.5, 0)
			nL.Position = UDim2.new(0, 10, 0, 5)
			nL.BackgroundTransparency = 1
			nL.Font = Enum.Font.GothamMedium
			nL.TextColor3 = Color3.new(1, 1, 1)
			nL.TextSize = 14
			nL.TextXAlignment = Enum.TextXAlignment.Left

			local sL = Instance.new("TextLabel", row)
			sL.Name = "StatLabel"
			sL.Size = UDim2.new(0.6, 0, 0.4, 0)
			sL.Position = UDim2.new(0, 10, 0.5, 0)
			sL.BackgroundTransparency = 1
			sL.Font = Enum.Font.Gotham
			sL.TextColor3 = TEXT_DIM
			sL.TextSize = 12
			sL.TextXAlignment = Enum.TextXAlignment.Left

			local oL = Instance.new("TextLabel", row)
			oL.Name = "OVRBadge"
			oL.Size = UDim2.new(0, 35, 0, 35)
			oL.Position = UDim2.new(1, -45, 0, 7.5)
			oL.BackgroundColor3 = NEON
			oL.TextColor3 = DARK
			oL.Font = Enum.Font.GothamBold
			oL.TextSize = 16
			Instance.new("UICorner", oL).CornerRadius = UDim.new(0, 6)

			local click = Instance.new("TextButton", row)
			click.Name = "ClickBtn"
			click.Size = UDim2.new(1, 0, 1, 0)
			click.BackgroundTransparency = 1
			click.Text = ""
		end

		row.Name = "Player_" .. player.Name
		row.Visible = true
		row.LayoutOrder = i

		local nL = row:FindFirstChild("NameLabel")
		if nL then nL.Text = (player.NationalityFlag or "🏳️") .. " " .. player.Name end

		local sL = row:FindFirstChild("StatLabel")
		if sL then sL.Text = player.Position .. " • Age " .. player.Age .. " • 😊" .. (player.Morale or 80) end

		local oL = row:FindFirstChild("OVRBadge")
		if oL then oL.Text = tostring(player.OVR) end

		local click = row:FindFirstChild("ClickBtn")
		if click then
			click.MouseButton1Click:Connect(function()
				local pn = previewCard:FindFirstChild("PlayerName") or Instance.new("TextLabel", previewCard)
				pn.Name = "PlayerName" pn.Size = UDim2.new(1,0,0,40) pn.Position = UDim2.new(0,0,0,10)
				pn.BackgroundTransparency = 1 pn.Text = player.Name:upper() pn.TextColor3 = NEON
				pn.Font = Enum.Font.Oswald pn.TextSize = 24

				local statHolder = previewCard:FindFirstChild("StatHolder") or Instance.new("Frame", previewCard)
				statHolder.Name = "StatHolder" statHolder.Size = UDim2.new(1,-20,0,180) statHolder.Position = UDim2.new(0,10,0,60)
				statHolder.BackgroundTransparency = 1
				local gl = statHolder:FindFirstChildOfClass("UIGridLayout") or Instance.new("UIGridLayout", statHolder)
				gl.CellSize = UDim2.new(0.48,0,0,45) gl.CellPadding = UDim2.new(0.04,0,0,8)

				local stats = player.Stats or {Shooting=60,Passing=60,Dribbling=60,Defending=60,Pace=60,Physical=60}
				for _, attr in ipairs({"Shooting","Passing","Dribbling","Defending","Pace","Physical"}) do
					local f = statHolder:FindFirstChild(attr) or Instance.new("Frame", statHolder)
					f.Name = attr f.BackgroundColor3 = SURFACE2 Instance.new("UICorner", f)
					local bl = f:FindFirstChild("Label") or Instance.new("TextLabel", f)
					bl.Name = "Label" bl.Size = UDim2.new(1,0,0.5,0) bl.BackgroundTransparency = 1 bl.TextColor3 = Color3.new(0.7,0.7,0.7)
					bl.Text = attr:upper() bl.Font = Enum.Font.GothamMedium bl.TextSize = 10
					local bv = f:FindFirstChild("Val") or Instance.new("TextLabel", f)
					bv.Name = "Val" bv.Size = UDim2.new(1,0,0.5,0) bv.Position = UDim2.new(0,0,0.5,0)
					bv.BackgroundTransparency = 1 bv.TextColor3 = Color3.new(1,1,1)
					bv.Text = tostring(stats[attr] or player.OVR) bv.Font = Enum.Font.GothamBold bv.TextSize = 16
				end

				-- Viewport
				local vp = setupViewport(previewCard)

				TweenWrapper:PlaySound("click")
			end)
		end
	end
	if playerList:IsA("ScrollingFrame") then playerList.CanvasSize = UDim2.new(0,0,0,#sorted*55) end
end

-- ================================================
-- 🎓 ACADEMY TAB (Pooled)
-- ================================================
function refreshAcademy()
	local data = fetchClubData()
	if not data then return end

	local panel = ContentArea:FindFirstChild("Panel_Academy")
	if not panel then return end
	purgeOldUI(panel)
	getOrCreateHeader(panel, "Youth Academy")

	local listCard = getOrCreateCard(panel, "YouthListCard", UDim2.new(1, -20, 1, -70), UDim2.new(0, 10, 0, 60))
	local list = listCard:FindFirstChild("YouthList")
	if not list then
		list = Instance.new("ScrollingFrame", listCard)
		list.Name = "YouthList"
		list.Size = UDim2.new(1, -20, 1, -45)
		list.Position = UDim2.new(0, 10, 0, 40)
		list.BackgroundTransparency = 1
		list.ScrollBarThickness = 2
		local l = Instance.new("UIListLayout", list)
		l.Padding = UDim.new(0, 10)
	end

	local ok, youth = pcall(function() return GetAcademyData:InvokeServer() end)
	if not ok or not youth then return end

	-- Return to pool
	if not ListPool["YouthRow"] then ListPool["YouthRow"] = {} end
	for _, child in ipairs(list:GetChildren()) do
		if child:IsA("Frame") then
			child.Visible = false
			table.insert(ListPool["YouthRow"], child)
		end
	end

	for i, p in ipairs(youth) do
		local row = getFromPool(ListPool["YouthRow"], list)
		if not row then
			row = Instance.new("Frame")
			row.Size = UDim2.new(1, -8, 0, 55)
			row.BackgroundColor3 = SURFACE
			row.BackgroundTransparency = 0.6
			Instance.new("UICorner", row).CornerRadius = UDim.new(0, 8)

			local nL = Instance.new("TextLabel", row)
			nL.Name = "NameLabel"
			nL.Size = UDim2.new(0.6, 0, 0.5, 0)
			nL.Position = UDim2.new(0, 10, 0, 5)
			nL.BackgroundTransparency = 1
			nL.Font = Enum.Font.GothamMedium
			nL.TextColor3 = Color3.new(1, 1, 1)
			nL.TextSize = 14
			nL.TextXAlignment = Enum.TextXAlignment.Left

			local pB = Instance.new("TextButton", row)
			pB.Name = "PromoteBtn"
			pB.Size = UDim2.new(0, 100, 0, 35)
			pB.Position = UDim2.new(1, -110, 0.5, -17)
			pB.BackgroundColor3 = NEON
			pB.TextColor3 = DARK
			pB.Font = Enum.Font.GothamBold
			pB.TextSize = 12
			pB.Text = "PROMOTE"
			Instance.new("UICorner", pB).CornerRadius = UDim.new(0, 6)
		end

		row.Name = "Youth_" .. p.Name
		row.Visible = true
		row.LayoutOrder = i

		local nL = row:FindFirstChild("NameLabel")
		if nL then nL.Text = p.Name .. " (" .. p.Position .. ")" end

		local pB = row:FindFirstChild("PromoteBtn")
		if pB then
			pB.MouseButton1Click:Connect(function()
				local success = PromoteYouth:InvokeServer(p.ID)
				if success then refreshAcademy() end
			end)
		end
	end
end

function refreshScouting()
	local data = fetchClubData()
	if not data then return end

	local panel = ContentArea:FindFirstChild("Panel_Scouting")
	if not panel then return end
	purgeOldUI(panel)
	getOrCreateHeader(panel, "Scouting")

	local missions = {"Local", "Regional", "Global"}
	local costs    = {Local=50000, Regional=250000, Global=1000000}

	-- Mission Cards
	for i, mType in ipairs(missions) do
		local card = getOrCreateCard(panel, mType .. "ScoutCard", UDim2.new(0.3, -10, 0, 160), UDim2.new((i-1)*0.33, 10, 0, 60))
		local priceL = card:FindFirstChild("Price") or Instance.new("TextLabel", card)
		priceL.Name = "Price"
		priceL.Size = UDim2.new(1,0,0,30)
		priceL.Position = UDim2.new(0,0,0,40)
		priceL.BackgroundTransparency = 1
		priceL.TextColor3 = TEXT_DIM
		priceL.Text = fmtMoney(costs[mType])
		priceL.Font = Enum.Font.GothamMedium

		local startBtn = card:FindFirstChild("StartBtn") or Instance.new("TextButton", card)
		startBtn.Name = "StartBtn"
		startBtn.Size = UDim2.new(0.8, 0, 0, 40)
		startBtn.Position = UDim2.new(0.1, 0, 1, -55)
		startBtn.BackgroundColor3 = (data.Budget >= costs[mType]) and NEON or SURFACE
		startBtn.TextColor3 = DARK
		startBtn.Text = "START MISSION"
		startBtn.Font = Enum.Font.GothamBold
		Instance.new("UICorner", startBtn)

		startBtn.MouseButton1Click:Connect(function()
			local ok, res = pcall(function() return SendScout:InvokeServer(mType) end)
			if ok then refreshScouting() end
		end)
	end

	-- Result List
	local resultCard = getOrCreateCard(panel, "ScoutResultsCard", UDim2.new(1, -20, 1, -240), UDim2.new(0, 10, 0, 230))
	local list = resultCard:FindFirstChild("ResultList")
	if not list then
		list = Instance.new("ScrollingFrame", resultCard)
		list.Name = "ResultList"
		list.Size = UDim2.new(1, -20, 1, -45)
		list.Position = UDim2.new(0, 10, 0, 40)
		list.BackgroundTransparency = 1
		list.CanvasSize = UDim2.new(0,0,0,1000)
		Instance.new("UIListLayout", list).Padding = UDim.new(0, 5)
	end

	local allPlayers = {}
	for _, m in ipairs(data.ScoutMissions or {}) do
		if m.Status == "COMPLETE" and m.Results then
			for _, p in ipairs(m.Results) do table.insert(allPlayers, p) end
		end
	end
	populateScoutResults(list, allPlayers)
end

function populateScoutResults(list, players)
	if not list then return end

	if not ListPool["ScoutRow"] then ListPool["ScoutRow"] = {} end
	for _, child in ipairs(list:GetChildren()) do
		if child:IsA("Frame") then
			child.Visible = false
			table.insert(ListPool["ScoutRow"], child)
		end
	end

	for i, p in ipairs(players) do
		local row = getFromPool(ListPool["ScoutRow"], list)
		if not row then
			row = Instance.new("Frame")
			row.Size = UDim2.new(1,-8,0,50)
			row.BackgroundColor3 = SURFACE2
			row.BackgroundTransparency = 0.5
			Instance.new("UICorner", row).CornerRadius = UDim.new(0,8)

			local t = Instance.new("TextLabel", row)
			t.Name = "Label"
			t.Size = UDim2.new(1,-10,1,0)
			t.Position = UDim2.new(0,5,0,0)
			t.BackgroundTransparency = 1
			t.Font = Enum.Font.Gotham
			t.TextColor3 = Color3.new(1, 1, 1)
			t.TextSize = 13
			t.TextXAlignment = Enum.TextXAlignment.Left
		end

		row.Visible = true
		local lbl = row:FindFirstChild("Label")
		if lbl then
			lbl.Text = p.Name .. " • OVR " .. p.OVR .. " • " .. fmtMoney(p.Value)
		end
	end
end

-- ================================================
-- 💰 FINANCES TAB (AAA Smoothing)
-- ================================================
function refreshFinances()
	local data = fetchClubData()
	if not data then return end

	local panel = ContentArea:FindFirstChild("Panel_Finances")
	if not panel then return end
	purgeOldUI(panel)
	local scrollContent = panel:FindFirstChild("ScrollContent")
	if not scrollContent then
		scrollContent = Instance.new("ScrollingFrame", panel)
		scrollContent.Name = "ScrollContent"
		scrollContent.Size = UDim2.new(1, -20, 1, -60)
		scrollContent.Position = UDim2.new(0, 10, 0, 60)
		scrollContent.BackgroundTransparency = 1
		scrollContent.CanvasSize = UDim2.new(0, 0, 0, 800) -- Room for growth
		scrollContent.ScrollBarThickness = 2
		scrollContent.BorderSizePixel = 0
	end

	-- Move Balance and Facilities DOWN to make room for Purchased Players at Top
	local boughtCard = getOrCreateCard(scrollContent, "PurchasedPlayersCard", UDim2.new(1, -10, 0, 180), UDim2.new(0, 5, 0, 10))
	local balCard    = getOrCreateCard(scrollContent, "BalanceCard", UDim2.new(1, -10, 0, 100), UDim2.new(0, 5, 0, 200))
	local facCard    = getOrCreateCard(scrollContent, "FacilitiesCard", UDim2.new(1, -10, 0, 250), UDim2.new(0, 5, 0, 310))

	-- Populate Purchased Players
	local bList = boughtCard:FindFirstChild("List") or Instance.new("UIListLayout", boughtCard)
	-- Logic for transfer history goes here...

	-- Balance Header (Smooth CountUp)
	local balLbl = balCard:FindFirstChild("BalanceValue") or Instance.new("TextLabel", balCard)
	balLbl.Name = "BalanceValue"
	balLbl.Size = UDim2.new(1,0,0,50)
	balLbl.Position = UDim2.new(0,0,0,40)
	balLbl.BackgroundTransparency = 1
	balLbl.Font = Enum.Font.Oswald
	balLbl.TextColor3 = Color3.new(1,1,1)
	balLbl.TextSize = 32
	TweenWrapper:CountUp(balLbl, 0, data.Budget, 1.0, "£")

	-- Populate Purchased Players
	local bList = boughtCard:FindFirstChild("List") or Instance.new("ScrollingFrame", boughtCard)
	if not bList:IsA("ScrollingFrame") then
		bList = Instance.new("ScrollingFrame", boughtCard)
		bList.Name = "List"
		bList.Size = UDim2.new(1,-20,1,-40)
		bList.Position = UDim2.new(0,10,0,35)
		bList.BackgroundTransparency = 1
		bList.CanvasSize = UDim2.new(0,0,0,500)
		Instance.new("UIListLayout", bList).Padding = UDim.new(0,5)
	end

	-- Populate from data (AI Signings / Player Signings)
	if not ListPool["FinanceRow"] then ListPool["FinanceRow"] = {} end
	for _, child in ipairs(bList:GetChildren()) do
		if child:IsA("Frame") then child.Visible = false table.insert(ListPool["FinanceRow"], child) end
	end

	for i, entry in ipairs(data.TransferHistory or {}) do
		local row = getFromPool(ListPool["FinanceRow"], bList)
		if not row then
			row = Instance.new("Frame", bList)
			row.Size = UDim2.new(1, 0, 0, 30)
			row.BackgroundTransparency = 0.8
			row.BackgroundColor3 = SURFACE
			Instance.new("UICorner", row).CornerRadius = UDim.new(0,6)
			local l = Instance.new("TextLabel", row)
			l.Name = "Label"
			l.Size = UDim2.new(1,-10,1,0)
			l.Position = UDim2.new(0,5,0,0)
			l.BackgroundTransparency = 1
			l.Font = Enum.Font.Gotham
			l.TextColor3 = Color3.new(1,1,1)
			l.TextSize = 12
		end
		row.Visible = true
		row:FindFirstChild("Label").Text = entry.Player .. ": " .. entry.From .. " ➔ " .. entry.To .. " (" .. fmtMoney(entry.Price) .. ")"
	end

	-- Facility levels
	local facilities = data.Facilities or {Training=1,Youth=1,Recovery=1}
	local listLayout = facCard:FindFirstChild("UIListLayout") or Instance.new("UIListLayout", facCard)
	listLayout.Padding = UDim.new(0, 10)
	listLayout.VerticalAlignment = Enum.VerticalAlignment.Center

	for _, fType in ipairs({"Training","Youth","Recovery"}) do
		local card = facCard:FindFirstChild(fType .. "Row") or Instance.new("Frame", facCard)
		card.Name = fType .. "Row"
		card.Size = UDim2.new(1, -20, 0, 60)
		card.BackgroundTransparency = 1

		local n = card:FindFirstChild("Label") or Instance.new("TextLabel", card)
		n.Name = "Label"
		n.Size = UDim2.new(0.4, 0, 1, 0)
		n.Position = UDim2.new(0, 10, 0, 0)
		n.Text = fType:upper() .. " LVL " .. (facilities[fType] or 1)
		n.TextColor3 = Color3.new(1,1,1)
		n.BackgroundTransparency = 1
		n.Font = Enum.Font.GothamMedium
		n.TextXAlignment = Enum.TextXAlignment.Left
	end

	-- Transfer History Card (New)
	local histCard = getOrCreateCard(panel, "TransferLedger", UDim2.new(1, -20, 0, 150), UDim2.new(0, 10, 0, 380))
	local list = histCard:FindFirstChild("List") or Instance.new("ScrollingFrame", histCard)
	list.Name = "List"
	list.Size = UDim2.new(1, -20, 1, -45)
	list.Position = UDim2.new(0, 10, 0, 40)
	list.BackgroundTransparency = 1
	list.CanvasSize = UDim2.new(0,0,0,500)
	list.ScrollBarThickness = 2
	local l = list:FindFirstChildOfClass("UIListLayout") or Instance.new("UIListLayout", list)
	l.Padding = UDim.new(0, 5)

	for i, entry in ipairs(data.TransferHistory or {}) do
		if i > 5 then break end
		local r = Instance.new("Frame", list)
		r.Size = UDim2.new(1,0,0,25) r.BackgroundTransparency = 1
		local n = Instance.new("TextLabel", r)
		n.Size = UDim2.new(1,0,1,0) n.BackgroundTransparency = 1
		n.Font = Enum.Font.Gotham n.TextSize = 11 n.TextColor3 = Color3.new(0.8,0.8,0.8)
		n.Text = entry.Player .. ": " .. entry.From .. " ➔ " .. entry.To
	end
end

-- ================================================
-- 📣 SPONSORS TAB (Smooth)
-- ================================================
function refreshSponsors()
	local data = fetchClubData()
	if not data then return end

	local panel = ContentArea:FindFirstChild("Panel_Sponsors")
	if not panel then return end
	purgeOldUI(panel)
	getOrCreateHeader(panel, "ACTIVE SPONSORSHIPS")

	for i, spon in ipairs(data.Sponsors or {}) do
		local card = getOrCreateCard(panel, "SponCard_" .. i, UDim2.new(0.3, -10, 0.8, 0), UDim2.new((i-1)*0.33, 10, 0, 60))
		local name = spon.Name:upper()
		local typeTxt = spon.Type:gsub("_", " "):upper()

		local nL = card:FindFirstChild("Name") or Instance.new("TextLabel", card)
		nL.Name = "Name" nL.Size = UDim2.new(1,0,0,30) nL.Position = UDim2.new(0,0,0,15)
		nL.BackgroundTransparency = 1 nL.Text = name nL.TextColor3 = Color3.new(1,1,1) nL.Font = Enum.Font.GothamBold nL.TextSize = 16

		local tL = card:FindFirstChild("Type") or Instance.new("TextLabel", card)
		tL.Name = "Type" tL.Size = UDim2.new(1,0,0,20) tL.Position = UDim2.new(0,0,0,45)
		tL.BackgroundTransparency = 1 tL.Text = typeTxt tL.TextColor3 = NEON tL.Font = Enum.Font.GothamMedium tL.TextSize = 11

		local b = card:FindFirstChild("ClaimBtn") or Instance.new("TextButton", card)
		b.Name = "ClaimBtn" b.Size = UDim2.new(0.8, 0, 0, 50) b.Position = UDim2.new(0.1, 0, 0.8, -10)
		b.BackgroundColor3 = (spon.Progress >= spon.GoalAmount and not spon.Claimed) and NEON or SURFACE2
		b.TextColor3 = (spon.Progress >= spon.GoalAmount and not spon.Claimed) and DARK or TEXT_DIM
		b.Text = spon.Claimed and "CLAIMED" or "CLAIM REWARD"
		b.Font = Enum.Font.GothamBold b.TextSize = 14 Instance.new("UICorner", b)

		b.MouseButton1Click:Connect(function()
			ClaimSponsor:InvokeServer(spon.Name)
			refreshSponsors()
		end)
	end
end

function refreshMarket()
	local data = fetchClubData()
	if not data then return end

	local panel = ContentArea:FindFirstChild("Panel_Market")
	if not panel then return end
	purgeOldUI(panel)
	getOrCreateHeader(panel, "Transfer Market")

	local marketCard = getOrCreateCard(panel, "MarketListCard", UDim2.new(0.6, -15, 1, -75), UDim2.new(0, 10, 0, 65))
	local detailCard = getOrCreateCard(panel, "MarketDetailCard", UDim2.new(0.4, -15, 1, -75), UDim2.new(0.6, 5, 0, 65))

	local list = marketCard:FindFirstChild("List")
	if not list then
		list = Instance.new("ScrollingFrame", marketCard)
		list.Name = "List"
		list.Size = UDim2.new(1, -20, 1, -45)
		list.Position = UDim2.new(0, 10, 0, 40)
		list.BackgroundTransparency = 1
		list.CanvasSize = UDim2.new(0,0,0,1000)
		list.ScrollBarThickness = 2
		local l = Instance.new("UIListLayout", list)
		l.Padding = UDim.new(0, 6)
	end

	local ok, marketPlayers = pcall(function() return GetTransferMarket:InvokeServer() end)
	if ok and marketPlayers then
		if not ListPool["MarketRow"] then ListPool["MarketRow"] = {} end
		for _, child in ipairs(list:GetChildren()) do
			if child:IsA("Frame") then child.Visible = false table.insert(ListPool["MarketRow"], child) end
		end

		for i, p in ipairs(marketPlayers) do
			local row = getFromPool(ListPool["MarketRow"], list)
			if not row then
				row = Instance.new("Frame")
				row.Size = UDim2.new(1, 0, 0, 45)
				row.BackgroundColor3 = Color3.new(0,0,0)
				row.BackgroundTransparency = 0.8
				Instance.new("UICorner", row).CornerRadius = UDim.new(0, 8)

				local n = Instance.new("TextLabel", row)
				n.Name = "Name"
				n.Size = UDim2.new(0.5, 0, 1, 0)
				n.Position = UDim2.new(0, 10, 0, 0)
				n.BackgroundTransparency = 1
				n.Font = Enum.Font.GothamMedium
				n.TextColor3 = Color3.new(1,1,1)
				n.TextXAlignment = Enum.TextXAlignment.Left

				local b = Instance.new("TextButton", row)
				b.Name = "ViewBtn"
				b.Size = UDim2.new(1,0,1,0)
				b.BackgroundTransparency = 1
				b.Text = ""
			end
			row.Visible = true
			row:FindFirstChild("Name").Text = p.Name .. " (" .. p.OVR .. ")"

			local btn = row:FindFirstChild("ViewBtn")
			if btn then
				btn.MouseButton1Click:Connect(function()
					local detName = detailCard:FindFirstChild("PlayerName") or Instance.new("TextLabel", detailCard)
					detName.Name = "PlayerName"
					detName.Size = UDim2.new(1, -20, 0, 40)
					detName.Position = UDim2.new(0, 10, 0, 35)
					detName.BackgroundTransparency = 1
					detName.Font = Enum.Font.Oswald
					detName.TextSize = 24
					detName.TextColor3 = NEON
					detName.Text = p.Name:upper()

					local buyBtn = detailCard:FindFirstChild("BuyBtn") or Instance.new("TextButton", detailCard)
					buyBtn.Name = "BuyBtn"
					buyBtn.Size = UDim2.new(0.8, 0, 0, 40)
					buyBtn.Position = UDim2.new(0.1, 0, 1, -60)
					buyBtn.BackgroundColor3 = NEON
					buyBtn.TextColor3 = DARK
					buyBtn.Font = Enum.Font.GothamBold
					buyBtn.Text = "SIGN PLAYER • " .. fmtMoney(p.Value or 500000)
					Instance.new("UICorner", buyBtn)
				end)
			end
		end
	end
end

-- ================================================
-- ⚙️ TACTICS TAB
-- ================================================
function refreshTactics()
	local data = fetchClubData()
	if not data then return end

	local panel = ContentArea:FindFirstChild("Panel_Tactics")
	if not panel then return end
	purgeOldUI(panel)
	getOrCreateHeader(panel, "TEAM TACTICS")

	local pitchCard = getOrCreateCard(panel, "PitchCard", UDim2.new(0.7, -15, 1, -70), UDim2.new(0, 10, 0, 60))
	local settingCard = getOrCreateCard(panel, "SettingsCard", UDim2.new(0.3, -15, 1, -70), UDim2.new(0.7, 5, 0, 60))

	-- ⚽ HORIZONTAL PITCH RENDER
	local pitch = pitchCard:FindFirstChild("Pitch")
	if not pitch then
		pitch = Instance.new("Frame", pitchCard)
		pitch.Name = "Pitch" pitch.Size = UDim2.new(0.9, 0, 0.8, 0) pitch.Position = UDim2.new(0.05, 0, 0.1, 0)
		pitch.BackgroundColor3 = Color3.fromRGB(30, 80, 40) pitch.BorderSizePixel = 0
		Instance.new("UICorner", pitch)
		local stroke = Instance.new("UIStroke", pitch) stroke.Color = Color3.new(1,1,1) stroke.Thickness = 2 stroke.Transparency = 0.4
		local grad = Instance.new("UIGradient", pitch)
		grad.Color = ColorSequence.new(Color3.fromRGB(40,100,50), Color3.fromRGB(20,60,30))

		-- Markers
		local function line(s, p)
			local f = Instance.new("Frame", pitch) f.Size = s f.Position = p
			f.BackgroundColor3 = Color3.new(1,1,1) f.BackgroundTransparency = 0.5 f.BorderSizePixel = 0
		end
		line(UDim2.new(0,2,1,0), UDim2.new(0.5,0,0,0)) -- Halfway
		local center = Instance.new("Frame", pitch) center.Size = UDim2.new(0,70,0,70) center.Position = UDim2.new(0.5,-35,0.5,-35)
		center.BackgroundTransparency = 1 local s = Instance.new("UIStroke", center) s.Color = Color3.new(1,1,1) s.Thickness = 2
		Instance.new("UICorner", center).CornerRadius = UDim.new(1,0)
	end

	local formations = {
		["4-4-2"] = {{0.05,0.5}, {0.3,0.2}, {0.3,0.4}, {0.3,0.6}, {0.3,0.8}, {0.5,0.2}, {0.5,0.4}, {0.5,0.6}, {0.5,0.8}, {0.8,0.4}, {0.8,0.6}},
		["4-3-3"] = {{0.05,0.5}, {0.25,0.2}, {0.25,0.4}, {0.25,0.6}, {0.25,0.8}, {0.5,0.3}, {0.5,0.5}, {0.5,0.7}, {0.8,0.2}, {0.8,0.5}, {0.8,0.8}},
		["4-2-3-1"] = {{0.05,0.5}, {0.2,0.2}, {0.2,0.4}, {0.2,0.6}, {0.2,0.8}, {0.45,0.4}, {0.45,0.6}, {0.7,0.2}, {0.7,0.5}, {0.7,0.8}, {0.9,0.5}},
		["3-5-2"] = {{0.05,0.5}, {0.2,0.3}, {0.2,0.5}, {0.2,0.7}, {0.5,0.1}, {0.5,0.3}, {0.5,0.5}, {0.5,0.7}, {0.5,0.9}, {0.8,0.4}, {0.8,0.6}}
	}

	local currentForm = formations[data.Formation or "4-3-3"] or formations["4-4-2"]

	for i = 1, 11 do
		local icon = pitch:FindFirstChild("Pos_" .. i)
		if not icon then
			icon = Instance.new("Frame", pitch) icon.Name = "Pos_" .. i
			icon.Size = UDim2.new(0, 36, 0, 36) icon.AnchorPoint = Vector2.new(0.5, 0.5)
			icon.BackgroundColor3 = NEON Instance.new("UICorner", icon).CornerRadius = UDim.new(1, 0)
			local l = Instance.new("TextLabel", icon) l.Size = UDim2.new(1, 0, 1, 0) l.BackgroundTransparency = 1
			l.Text = tostring(i) l.TextColor3 = DARK l.Font = Enum.Font.GothamBold l.TextSize = 12
		end
		tween(icon, { Position = UDim2.new(currentForm[i][1], 0, currentForm[i][2], 0) }, 0.6, Enum.EasingStyle.Back)
	end

	local picker = settingCard:FindFirstChild("FormationPicker") or Instance.new("Frame", settingCard)
	picker.Name = "FormationPicker" picker.Size = UDim2.new(1, -20, 0, 200) picker.Position = UDim2.new(0, 10, 0, 50)
	picker.BackgroundTransparency = 1
	local gl = picker:FindFirstChildOfClass("UIGridLayout") or Instance.new("UIGridLayout", picker)
	gl.CellSize = UDim2.new(0, 85, 0, 35) gl.CellPadding = UDim2.new(0,8,0,8)

	for formName, _ in pairs(formations) do
		local b = picker:FindFirstChild("Form_"..formName) or Instance.new("TextButton", picker)
		b.Name = "Form_"..formName b.Text = formName b.BackgroundColor3 = (data.Formation == formName) and NEON or SURFACE2
		b.TextColor3 = (data.Formation == formName) and DARK or Color3.new(1,1,1)
		b.Font = Enum.Font.GothamBold b.TextSize = 11 Instance.new("UICorner", b)
		b.MouseButton1Click:Connect(function()
			SetTactics:InvokeServer(formName, data.Playstyle, data.StartingXI)
			refreshTactics()
		end)
	end

	-- 📋 SQUAD DRAFTER (XI SWAP)
	local drafter = settingCard:FindFirstChild("Drafter") or Instance.new("ScrollingFrame", settingCard)
	drafter.Name = "Drafter" drafter.Size = UDim2.new(1, -20, 1, -250) drafter.Position = UDim2.new(0, 10, 0, 240)
	drafter.BackgroundTransparency = 1 drafter.CanvasSize = UDim2.new(0,0,0,800) drafter.ScrollBarThickness = 1
	local dl = drafter:FindFirstChildOfClass("UIListLayout") or Instance.new("UIListLayout", drafter)
	dl.Padding = UDim.new(0, 5)

	local selectedForSwap = nil
	local roster = {}
	if data and data.Roster then
		for _, p in pairs(data.Roster) do table.insert(roster, p) end
		table.sort(roster, function(a,b) return a.OVR > b.OVR end)
	end

	for i, p in ipairs(roster) do
		local r = drafter:FindFirstChild(p.ID) or Instance.new("TextButton", drafter)
		r.Name = p.ID r.Size = UDim2.new(1, 0, 0, 30) r.BackgroundColor3 = SURFACE2
		r.Text = p.Name .. " (" .. p.OVR .. ")" r.TextColor3 = Color3.new(1,1,1)
		r.Font = Enum.Font.GothamMedium r.TextSize = 10 Instance.new("UICorner", r)

		r.MouseButton1Click:Connect(function()
			selectedForSwap = p.ID
			for _, other in ipairs(drafter:GetChildren()) do if other:isA("TextButton") then other.BackgroundColor3 = SURFACE2 end end
			r.BackgroundColor3 = NEON r.TextColor3 = DARK
		end)
	end

	-- Apply Swap to Pitch
	for i = 1, 11 do
		local icon = pitch:FindFirstChild("Pos_" .. i)
		if icon then
			local btn = icon:FindFirstChild("SwapBtn") or Instance.new("TextButton", icon)
			btn.Name = "SwapBtn" btn.Size = UDim2.new(1,0,1,0) btn.BackgroundTransparency = 1 btn.Text = ""
			btn.MouseButton1Click:Connect(function()
				if selectedForSwap then
					local newXI = data.StartingXI or {}
					newXI[tostring(i)] = selectedForSwap
					SetTactics:InvokeServer(data.Formation, data.Playstyle, newXI)
					refreshTactics()
					selectedForSwap = nil
				end
			end)

			-- Show name if slot is occupied
			local currentID = (data.StartingXI and data.StartingXI[tostring(i)])
			local pData = currentID and data.Roster[currentID]
			local nameLbl = icon:FindFirstChild("NameLbl") or Instance.new("TextLabel", icon)
			nameLbl.Name = "NameLbl" nameLbl.Size = UDim2.new(2,0,0,15) nameLbl.Position = UDim2.new(-0.5,0,1,2)
			nameLbl.BackgroundTransparency = 1 nameLbl.TextColor3 = Color3.new(1,1,1)
			nameLbl.Font = Enum.Font.GothamMedium nameLbl.TextSize = 8
			nameLbl.Text = pData and pData.Name:split(" ")[#pData.Name:split(" ")]:upper() or "VACANT"
		end
	end
end

-- ================================================
-- 🌍 WORLD TAB (Pooled)
-- ================================================
function refreshWorld()
	local data = fetchClubData()
	if not data then return end

	local panel = ContentArea:FindFirstChild("Panel_World")
	if not panel then return end
	purgeOldUI(panel)
	getOrCreateHeader(panel, "GLOBAL FOOTBALL HUB")

	local newsCard  = getOrCreateCard(panel, "NewsCard", UDim2.new(0.6, -15, 0.45, -10), UDim2.new(0, 10, 0, 60))
	local rankCard  = getOrCreateCard(panel, "RankingsCard", UDim2.new(0.4, -15, 0.5, -10), UDim2.new(0.6, 5, 0, 60))
	local mapCard   = getOrCreateCard(panel, "WorldMapCard", UDim2.new(1, -20, 0.4, -20), UDim2.new(0, 10, 0, 310))

	-- Championship Buttons
	local cHolder = rankCard:FindFirstChild("ChampButtons") or Instance.new("Frame", rankCard)
	cHolder.Name = "ChampButtons" cHolder.Size = UDim2.new(1, -20, 0, 35) cHolder.Position = UDim2.new(0, 10, 0, 40)
	cHolder.BackgroundTransparency = 1
	local gl = cHolder:FindFirstChildOfClass("UIGridLayout") or Instance.new("UIGridLayout", cHolder)
	gl.CellSize = UDim2.new(0, 100, 1, 0) gl.CellPadding = UDim2.new(0,5,0,0)

	local champs = {"DOMESTIC LEAGUE", "CONTINENTAL CUP", "GLOBAL TROPHY"}
	for _, c in ipairs(champs) do
		local b = cHolder:FindFirstChild(c) or Instance.new("TextButton", cHolder)
		b.Name = c b.Text = c b.BackgroundColor3 = SURFACE2 b.TextColor3 = Color3.new(1,1,1)
		b.Font = Enum.Font.GothamBold b.TextSize = 10 Instance.new("UICorner", b)
		b.MouseButton1Click:Connect(function()
			for _, other in ipairs(cHolder:GetChildren()) do if other:IsA("TextButton") then other.BackgroundColor3 = SURFACE2 end end
			b.BackgroundColor3 = NEON b.TextColor3 = DARK
		end)
	end

	-- NEWS logic
	local newsFeed = newsCard:FindFirstChild("NewsFeed") or Instance.new("ScrollingFrame", newsCard)
	newsFeed.Name = "NewsFeed" newsFeed.Size = UDim2.new(1, -24, 1, -50) newsFeed.Position = UDim2.new(0, 12, 0, 40)
	newsFeed.BackgroundTransparency = 1 newsFeed.CanvasSize = UDim2.new(0,0,0,1000) newsFeed.ScrollBarThickness = 1
	local l = newsFeed:FindFirstChildOfClass("UIListLayout") or Instance.new("UIListLayout", newsFeed)
	l.Padding = UDim.new(0, 8)

	local function updateRanks(scope, reg)
		local ok, standings = pcall(function() return GetLeagueData:InvokeServer(scope, reg, 1) end)
		if ok and standings then
			local rList = rankCard:FindFirstChild("RankList") or Instance.new("ScrollingFrame", rankCard)
			rList.Name = "RankList" rList.Size = UDim2.new(1,-20,1,-90) rList.Position = UDim2.new(0,10,0,80)
			rList.BackgroundTransparency = 1 rList.CanvasSize = UDim2.new(0,0,0,600)
			for _, child in ipairs(rList:GetChildren()) do if child:IsA("Frame") then child:Destroy() end end
			local layout = rList:FindFirstChildOfClass("UIListLayout") or Instance.new("UIListLayout", rList)
			layout.Padding = UDim.new(0,5)

			for i, club in ipairs(standings) do
				local row = Instance.new("Frame", rList) row.Size = UDim2.new(1,0,0,30) row.BackgroundTransparency = 0.8 row.BackgroundColor3 = SURFACE2
				local lb = Instance.new("TextLabel", row) lb.Size = UDim2.new(1,-10,1,0) lb.Position = UDim2.new(0,5,0,0) lb.BackgroundTransparency = 1
				lb.Font = Enum.Font.GothamMedium lb.TextSize = 11 lb.TextColor3 = Color3.new(1,1,1)
				lb.Text = i .. ". " .. club.Name .. " (" .. (club.LeagueStats.Points or 0) .. " pts) | GD: " .. (club.LeagueStats.GD or 0)
			end
		end
	end
	updateRanks("Domestic", "England")

	-- Championship Buttons Functional
	for _, c in ipairs(champs) do
		local b = cHolder:FindFirstChild(c)
		if b then
			b.MouseButton1Click:Connect(function()
				local scope = "Domestic"
				if c == "CONTINENTAL CUP" then scope = "Continental"
				elseif c == "GLOBAL TROPHY" then scope = "Global" end
				updateRanks(scope, data.Region)
			end)
		end
	end

	-- Map
	local regions = {"England", "Spain", "Germany", "France", "Italy"}
	for i, reg in ipairs(regions) do
		local btn = mapCard:FindFirstChild("Btn_" .. reg) or Instance.new("TextButton", mapCard)
		btn.Name = "Btn_" .. reg btn.Size = UDim2.new(0.18, 0, 0, 45) btn.Position = UDim2.new(0.02 + ((i-1)*0.19), 0, 0.45, 0)
		btn.BackgroundColor3 = SURFACE btn.TextColor3 = Color3.new(1,1,1) btn.Text = reg:upper()
		btn.Font = Enum.Font.GothamBold btn.TextSize = 11 Instance.new("UICorner", btn)
		btn.MouseButton1Click:Connect(function() updateRanks("Domestic", reg) end)
	end
end

-- ================================================
-- 🎮 MAIN MENU
-- ================================================
local playerBtn   = MainMenu:FindFirstChild("PlayerCareerBtn")
local managerBtn  = MainMenu:FindFirstChild("ManagerCareerBtn")
local settingsBtn = MainMenu:FindFirstChild("SettingsButton")

if playerBtn then
	playerBtn.MouseButton1Click:Connect(function()
		if UI:FindFirstChild("LockedToast") then return end
		local toast = Instance.new("TextLabel")
		toast.Name = "LockedToast"
		toast.Size = UDim2.new(0,520,0,80)
		toast.Position = UDim2.new(0.5,-260,0,-110)
		toast.BackgroundColor3 = DARK toast.TextColor3 = NEON
		toast.Text = "⚠️ Player Career Mode is temporarily unavailable.\nCheck back soon — try Manager Mode in the meantime!"
		toast.Font = Enum.Font.GothamMedium toast.TextSize = 14 toast.TextWrapped = true
		toast.ZIndex = 100 toast.BorderSizePixel = 0
		Instance.new("UICorner", toast).CornerRadius = UDim.new(0,10)
		local str = Instance.new("UIStroke", toast)
		str.Color = NEON str.Thickness = 1.5 str.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
		toast.Parent = UI
		tween(toast, { Position = UDim2.new(0.5,-260,0,30) }, 0.45, Enum.EasingStyle.Back)
		task.delay(4, function()
			local t2 = TweenService:Create(toast, TweenInfo.new(0.35, Enum.EasingStyle.Back, Enum.EasingDirection.In), { Position = UDim2.new(0.5,-260,0,-110) })
			t2:Play() t2.Completed:Connect(function() toast:Destroy() end)
		end)
	end)
end

if managerBtn then
	managerBtn.MouseButton1Click:Connect(function()
		if HasManagerCareer then
			PageManager:SwitchPage("ManagerDashboard", UIController)
			task.spawn(refreshOverview)
		else
			if ClubCreator then
				PageManager:SwitchPage("ClubCreator", UIController)
			else
				warn("[UI] ClubCreator page not found!")
			end
		end
	end)
end

if settingsBtn and SettingsPage then
	settingsBtn.MouseButton1Click:Connect(function()
		PageManager:SwitchPage("Settings", UIController)
	end)
end

-- ================================================
-- 🏟️ CLUB CREATOR WIZARD (Polished Styles)
-- ================================================
if ClubCreator then
	local Modal         = ClubCreator:FindFirstChild("Modal")
	local NameInput     = Modal and Modal:FindFirstChild("ClubNameInput")
	local ShortInput    = Modal and Modal:FindFirstChild("ShortNameInput")
	local ManagerInput  = Modal and Modal:FindFirstChild("ManagerNameInput")
	local RegionHolder  = Modal and Modal:FindFirstChild("RegionContainer")
	local DivisionHolder= Modal and Modal:FindFirstChild("DivisionContainer")
	local StartBtn      = Modal and Modal:FindFirstChild("StartCareerBtn")

	if StartBtn then
		StartBtn.TextColor3 = Color3.new(1,1,1) -- Switch to high-contrast white
		StartBtn.Font = Enum.Font.GothamBold
		StartBtn.TextScaled = true
		Instance.new("UITextSizeConstraint", StartBtn).MaxTextSize = 24
	end

	local selectedRegion   = "England"
	local selectedDivision = 10

	-- Kit color pickers
	local homeKitPick = Modal and Modal:FindFirstChild("HomeKitPicker")
	local awayKitPick = Modal and Modal:FindFirstChild("AwayKitPicker")
	local selectedHomeKit = { R=0, G=100, B=220 }
	local selectedAwayKit = { R=255, G=255, B=255 }

	if homeKitPick then
		for _, btn in ipairs(homeKitPick:GetChildren()) do
			if btn:IsA("TextButton") then
				btn.MouseButton1Click:Connect(function()
					local c = btn.BackgroundColor3
					selectedHomeKit = { R=math.floor(c.R*255), G=math.floor(c.G*255), B=math.floor(c.B*255) }
					for _, b in ipairs(homeKitPick:GetChildren()) do
						if b:IsA("TextButton") then b.BorderMode = Enum.BorderMode.Outline b.BorderSizePixel = 0 end
					end
					btn.BorderSizePixel = 3
				end)
			end
		end
	end

	if RegionHolder then
		local grid = RegionHolder:FindFirstChildOfClass("UIGridLayout") or Instance.new("UIGridLayout", RegionHolder)
		grid.CellSize = UDim2.new(0, 180, 0, 65)
		grid.CellPadding = UDim2.new(0, 15, 0, 15)
		grid.HorizontalAlignment = Enum.HorizontalAlignment.Center

		for _, btn in ipairs(RegionHolder:GetChildren()) do
			if btn:IsA("TextButton") then
				btn.TextColor3 = Color3.new(1,1,1) -- WHITE
				btn.Font = Enum.Font.GothamBold
				btn.TextScaled = true
				Instance.new("UITextSizeConstraint", btn).MaxTextSize = 18
				btn.MouseButton1Click:Connect(function()
					selectedRegion = btn.Text
					for _, b in ipairs(RegionHolder:GetChildren()) do
						if b:IsA("TextButton") then
							tween(b, { BackgroundColor3 = DARK, TextColor3 = TEXT_DIM }, 0.2)
							local s = b:FindFirstChildOfClass("UIStroke")
							if s then tween(s, { Color = Color3.fromRGB(60,60,75) }, 0.2) end
						end
					end
					tween(btn, { BackgroundColor3 = Color3.fromRGB(0,60,40), TextColor3 = NEON }, 0.2)
					local s = btn:FindFirstChildOfClass("UIStroke")
					if s then tween(s, { Color = NEON }, 0.2) end
				end)
			end
		end
	end

	if DivisionHolder then
		for _, btn in ipairs(DivisionHolder:GetChildren()) do
			if btn:IsA("TextButton") then
				btn.Size = UDim2.new(0, 180, 0, 60) -- BIGGER
				btn.TextColor3 = Color3.new(0,0,0) -- BLACK TEXT
				btn.Font = Enum.Font.GothamBold
				local divNum = tonumber(btn.Text) or tonumber(btn.Name:match("%d+")) or 10
				btn.MouseButton1Click:Connect(function()
					selectedDivision = divNum
					for _, b in ipairs(DivisionHolder:GetChildren()) do
						if b:IsA("TextButton") then tween(b, { BackgroundColor3 = SURFACE, TextColor3 = TEXT_DIM }, 0.2) end
					end
					tween(btn, { BackgroundColor3 = NEON, TextColor3 = DARK }, 0.2)
				end)
			end
		end
	end

	if StartBtn then
		StartBtn.MouseButton1Click:Connect(function()
			local cName  = (NameInput and NameInput.Text ~= "") and NameInput.Text or "Custom FC"
			local sName  = (ShortInput and ShortInput.Text ~= "") and ShortInput.Text or "CFC"
			local mName  = (ManagerInput and ManagerInput.Text) or ""

			StartBtn.Text = "⏳ VALIDATING..."
			StartBtn.BackgroundColor3 = SURFACE2

			local ok, success, msg = pcall(function() 
				return SetCareerMode:InvokeServer(cName, sName, mName, selectedRegion, selectedDivision, selectedHomeKit, selectedAwayKit, "Shield", "⚽")
			end)

			if not ok or not success then
				if StartBtn and StartBtn.Parent then
					StartBtn.Text = "❌ " .. tostring(msg or "Invalid name")
					StartBtn.BackgroundColor3 = RED
					task.delay(2.5, function()
						if StartBtn and StartBtn.Parent then
							StartBtn.Text = "START CAREER"
							StartBtn.BackgroundColor3 = NEON
						end
					end)
				end
				return
			end

			-- Success animation
			StartBtn.Text = "✅ CLUB CREATED!"
			tween(Modal, { Position = UDim2.new(0.5,-250,0,-700) }, 0.55, Enum.EasingStyle.Back, Enum.EasingDirection.In)
			HasManagerCareer = true
			task.wait(0.6)

			PageManager:SwitchPage("ManagerDashboard", UIController)
			switchTab("Overview")

			-- Reset modal for next time
			task.delay(1, function()
				Modal.Position = UDim2.new(0.5,-250,0.5,-300)
				StartBtn.Text = "START CAREER"
				StartBtn.BackgroundColor3 = NEON
			end)
		end)
	end
end

-- ================================================
-- ⚽ MATCH ENGINE
-- ================================================
local function refreshMatchHUD(data)
	if MatchSimPage then
		local sb = MatchSimPage:FindFirstChild("Scoreboard") or MatchSimPage:FindFirstChild("Scoreboard", true)
		if sb then sb.Text = data.Score.Home .. " — " .. data.Score.Away end
		local tl = MatchSimPage:FindFirstChild("TimerLabel") or MatchSimPage:FindFirstChild("Timer", true)
		if tl then tl.Text = math.floor(data.Minute or 0) .. "'" end

		-- ⚽ VISUAL PITCH ENGINE
		local field = MatchSimPage:FindFirstChild("SimPitch")
		if not field then
			field = Instance.new("Frame", MatchSimPage)
			field.Name = "SimPitch" field.Size = UDim2.new(0, 600, 0, 350)
			field.Position = UDim2.new(0.5, -300, 0.45, -175)
			field.BackgroundColor3 = Color3.fromRGB(20, 60, 30) Instance.new("UICorner", field)
			Instance.new("UIStroke", field).Color = Color3.new(1,1,1)

			-- Center Line
			local cl = Instance.new("Frame", field)
			cl.Size = UDim2.new(0, 2, 1, 0) cl.Position = UDim2.new(0.5, -1, 0, 0)
			cl.BackgroundColor3 = Color3.new(1,1,1) cl.BackgroundTransparency = 0.6
		end

		-- Render Player Dots (Simple placeholder logic for motion)
		for side, color in pairs({Home = NEON, Away = Color3.fromRGB(0, 150, 255)}) do
			for i = 1, 11 do
				local dotName = side .. "_" .. i
				local dot = field:FindFirstChild(dotName) or Instance.new("Frame", field)
				dot.Name = dotName dot.Size = UDim2.new(0, 10, 0, 10) dot.AnchorPoint = Vector2.new(0.5, 0.5)
				dot.BackgroundColor3 = color Instance.new("UICorner", dot).CornerRadius = UDim.new(1,0)

				-- Jitter movement for simulation feel
				local randX = (side == "Home") and math.random(10, 280) or math.random(320, 590)
				local randY = math.random(20, 330)
				tween(dot, { Position = UDim2.new(0, randX, 0, randY) }, 1.5)
			end
		end

		-- ⚽ BALL
		local ball = field:FindFirstChild("Ball") or Instance.new("Frame", field)
		ball.Name = "Ball" ball.Size = UDim2.new(0, 8, 0, 8) ball.BackgroundColor3 = Color3.new(1,1,1)
		Instance.new("UICorner", ball).CornerRadius = UDim.new(1,0)
		if data.Phase == "Chance" then
			local targetX = (data.Possession == "Home") and 590 or 10
			tween(ball, { Position = UDim2.new(0, targetX, 0, 175) }, 0.4)
		else
			tween(ball, { Position = UDim2.new(0, math.random(100, 500), 0, math.random(50, 300)) }, 1.0)
		end
	end
end

if MatchUpdate then
	MatchUpdate.OnClientEvent:Connect(function(data, eventType, msg)
		refreshMatchHUD(data)
		if msg then 
			local log = MatchSimPage:FindFirstChild("EventLog", true)
			if log then log.Text = msg:upper() end
		end

		if eventType == "MatchEnd" then
			MatchIsLive = false
			if KickOffBtn then KickOffBtn.Text = "⚽  PLAY MATCH" KickOffBtn.BackgroundColor3 = NEON end
		end
	end)
end

-- ================================================
-- 🔙 BACK BUTTONS
-- ================================================
for _, pageFrame in ipairs(PagesFolder:GetChildren()) do
	local backBtn = pageFrame:FindFirstChild("BackButton")
	if not backBtn and pageFrame.Name == "ManagerDashboard" then
		backBtn = TopBar and TopBar:FindFirstChild("BackButton")
	end

	if backBtn then
		backBtn.MouseButton1Click:Connect(function()
			PageManager:SwitchPage("MainMenu", UIController)
		end)
	end
end

-- ================================================
-- 🚀 BOOT SEQUENCE
-- ================================================
-- Set manager avatar on HUD
local HomeHUD = MatchSimPage:FindFirstChild("HomeManagerHUD")
if HomeHUD then
	pcall(function()
		local hs = HomeHUD:FindFirstChild("Headshot")
		if hs then hs.Image = Players:GetUserThumbnailAsync(Player.UserId, Enum.ThumbnailType.HeadShot, Enum.ThumbnailSize.Size420x420) end
	end)
	local textC = HomeHUD:FindFirstChild("TextContainer")
	if textC then
		local mgr = textC:FindFirstChild("ManagerName")
		if mgr then mgr.Text = "MGR: " .. Player.Name end
	end
end

-- Helper function used by OVR badges (avoids a closure issue)
function instance_new_corner(obj)
	Instance.new("UICorner", obj).CornerRadius = UDim.new(0,8)
end

-- ================================================
-- 📑 TAB SYSTEM (AAA Transitions)
-- ================================================
function switchTab(tabName)
	if not ContentArea then return end
	CurrentTab = tabName

	-- Color tab buttons
	for _, b in ipairs(TabBar:GetChildren()) do
		if b:IsA("TextButton") and b.Name:match("Tab_") then
			local isActive = (b.Name == "Tab_"..tabName)
			game:GetService("TweenService"):Create(b, TweenInfo.new(0.25), { 
				TextColor3 = isActive and NEON or TEXT_DIM 
			}):Play()
		end
	end

	-- Show/hide panels
	for _, panel in ipairs(ContentArea:GetChildren()) do
		if panel:IsA("CanvasGroup") then
			if panel.Name == "Panel_" .. tabName then
				panel.Visible = true
				TweenWrapper:FadeIn(panel, 0.4)
			else
				panel.Visible = false
			end
		end
	end

	-- Refresh data for the active tab
	task.spawn(function()
		local f = {
			Overview = refreshOverview, Squad = refreshSquad, Scouting = refreshScouting,
			Academy = refreshAcademy, Finances = refreshFinances, Sponsors = refreshSponsors,
			Tactics = refreshTactics, World = refreshWorld, Market = refreshMarket
		}
		if f[tabName] then f[tabName]() end
	end)
end

if TabBar then
	for _, btn in ipairs(TabBar:GetChildren()) do
		if btn:IsA("TextButton") and btn.Name:match("Tab_") then
			btn.MouseButton1Click:Connect(function()
				switchTab(btn.Name:sub(5))
			end)
		end
	end
end

-- ✅ FINAL BOOT
PageManager:SwitchPage("MainMenu", UIController)

-- AAA Background Polish
if ManagerDash then
	local grad = ManagerDash:FindFirstChild("UIGradient") or Instance.new("UIGradient", ManagerDash)
	grad.Color = ColorSequence.new({
		ColorSequenceKeypoint.new(0, Color3.fromRGB(15,15,22)),
		ColorSequenceKeypoint.new(1, Color3.fromRGB(5,5,10))
	})
	grad.Rotation = 45
end

-- Auto-apply Card Glow to Dashboard on Start
local dashPanel = ContentArea:FindFirstChild("Panel_Overview")
if dashPanel then
	for _, child in ipairs(dashPanel:GetChildren()) do
		if child:IsA("Frame") and child.Name:match("Card") then
			applyGlassStyle(child)
		end
	end
end

warn("✅ [UI] Manager Mode v3.0 Ready")

-- 🛡️ ADMIN PANEL SETUP
if ClubProfile and (ClubProfile.Role == "OWNER" or ClubProfile.Role == "ADMIN") then
	AdminPanel:Init(UI, ClubProfile.Role)

	-- Open with a key shortcut (Insert) or Button
	game:GetService("UserInputService").InputBegan:Connect(function(input, gpe)
		if not gpe and input.KeyCode == Enum.KeyCode.Insert then
			AdminPanel:Toggle()
		end
	end)
end

-- 📰 NEWS UPDATES
if NewsUpdate then
	NewsUpdate.OnClientEvent:Connect(function(headline)
		NewsTicker:Show(headline)
	end)
end

________________________________________________

UIController:

-- ================================================
-- UI CONTROLLER (Client/Core) v3.0
-- ================================================
-- Master UI logic handler. Manages AAA transitions
-- and global component behavior.
-- ================================================

local Core         = script.Parent
local TweenWrapper = require(Core:WaitForChild("TweenServiceWrapper"))

local UIController = {}

-- ================================================
-- PAGE TRANSITIONS
-- ================================================
function UIController:ShowPage(frame)
	if not frame then return end

	-- Automatic hover effects for all buttons on this page
	TweenWrapper:ApplyHoverToAll(frame)

	-- Page entry transition
	TweenWrapper:PageTransitionIn(frame, 0.4)
end

function UIController:ShowPageWithSlide(frame, direction)
	if not frame then return end

	-- Automatic hover effects
	TweenWrapper:ApplyHoverToAll(frame)

	-- Slide entry transition
	TweenWrapper:SlideIn(frame, direction or "up", 30, 0.4)
	TweenWrapper:FadeIn(frame, 0.4)

	TweenWrapper:PlaySound("whoosh")
end

function UIController:HidePage(frame)
	if not frame then return end
	TweenWrapper:PageTransitionOut(frame, 0.3)
end

-- ================================================
-- UNIVERSAL UI SETUP
-- ================================================
function UIController:Init(uiRoot)
	-- Listen for dynamic buttons added to UI
	uiRoot.DescendantAdded:Connect(function(desc)
		if desc:IsA("TextButton") then
			TweenWrapper:ButtonHover(desc)
		end
	end)

	-- Initialize existing buttons
	TweenWrapper:ApplyHoverToAll(uiRoot)
end

return UIController


______________________________________________

TwweenServiceWrapper:

-- ================================================
-- TWEEN SERVICE WRAPPER (Client/Core) v3.0
-- ================================================
-- AAA UI animation helpers. All animations use
-- Quad.Out at 0.25–0.35s per spec.
-- ================================================

local TweenService  = game:GetService("TweenService")
local RunService    = game:GetService("RunService")
local SoundService  = game:GetService("SoundService")

local TweenWrapper  = {}

-- ================================================
-- STANDARD SETTINGS
-- ================================================
local T_SHORT  = 0.25
local T_MED    = 0.30
local T_LONG   = 0.40
local STYLE    = Enum.EasingStyle.Quad
local DIR_OUT  = Enum.EasingDirection.Out
local NEON     = Color3.fromRGB(0, 255, 150)
local SURFACE2 = Color3.fromRGB(25, 25, 35)

local function makeTI(t, s, d)
	return TweenInfo.new(t or T_MED, s or STYLE, d or DIR_OUT)
end

-- ================================================
-- FADE IN / OUT (CanvasGroup)
-- ================================================
function TweenWrapper:FadeIn(element, duration)
	if not element then warn("[TW] FadeIn: element nil") return end
	duration = duration or T_MED
	element.Visible = true
	if element:IsA("CanvasGroup") then
		element.GroupTransparency = 1
		TweenService:Create(element, makeTI(duration), { GroupTransparency = 0 }):Play()
	end
end

function TweenWrapper:FadeOut(element, duration)
	if not element then return end
	duration = duration or T_SHORT
	if element:IsA("CanvasGroup") then
		local t = TweenService:Create(element, makeTI(duration), { GroupTransparency = 1 })
		t.Completed:Connect(function() element.Visible = false end)
		t:Play()
	else
		element.Visible = false
	end
end

-- ================================================
-- SLIDE IN (panel enters from direction)
-- direction: "up" | "down" | "left" | "right"
-- offset: pixel offset (default 30)
-- ================================================
function TweenWrapper:SlideIn(frame, direction, offset, duration)
	if not frame then return end
	offset   = offset   or 30
	duration = duration or T_MED
	local orig = frame.Position
	local startPos

	if direction == "up" then
		startPos = UDim2.new(orig.X.Scale, orig.X.Offset, orig.Y.Scale, orig.Y.Offset + offset)
	elseif direction == "down" then
		startPos = UDim2.new(orig.X.Scale, orig.X.Offset, orig.Y.Scale, orig.Y.Offset - offset)
	elseif direction == "left" then
		startPos = UDim2.new(orig.X.Scale, orig.X.Offset + offset, orig.Y.Scale, orig.Y.Offset)
	elseif direction == "right" then
		startPos = UDim2.new(orig.X.Scale, orig.X.Offset - offset, orig.Y.Scale, orig.Y.Offset)
	else
		startPos = UDim2.new(orig.X.Scale, orig.X.Offset, orig.Y.Scale, orig.Y.Offset + offset)
	end

	frame.Position = startPos
	frame.Visible  = true
	TweenService:Create(frame, makeTI(duration), { Position = orig }):Play()
end

-- ================================================
-- COUNT UP (smooth numeric animation for labels)
-- ================================================
function TweenWrapper:CountUp(label, fromVal, toVal, duration, prefix, suffix)
	if not label then return end
	duration = duration or T_LONG
	prefix   = prefix or ""
	suffix   = suffix or ""
	local startTime = tick()
	local range     = toVal - fromVal

	local conn
	conn = RunService.Heartbeat:Connect(function()
		local elapsed = tick() - startTime
		local t       = math.min(elapsed / duration, 1)
		-- Ease-out cubic
		local eased   = 1 - (1 - t)^3
		local current = math.floor(fromVal + range * eased)
		pcall(function() label.Text = prefix .. tostring(current) .. suffix end)
		if t >= 1 then
			conn:Disconnect()
			pcall(function() label.Text = prefix .. tostring(toVal) .. suffix end)
		end
	end)
end

-- ================================================
-- GLOW PULSE (repeating UIStroke neon glow)
-- ================================================
function TweenWrapper:GlowPulse(element, color)
	if not element then return end
	color = color or NEON
	local stroke = element:FindFirstChildOfClass("UIStroke")
	if not stroke then
		stroke = Instance.new("UIStroke", element)
		stroke.Thickness = 1.5
		stroke.Color     = color
	end
	stroke.Color = color

	local function pulse()
		TweenService:Create(stroke, makeTI(0.8, Enum.EasingStyle.Sine), {
			Transparency = 1
		}):Play()
		task.wait(0.85)
		TweenService:Create(stroke, makeTI(0.8, Enum.EasingStyle.Sine), {
			Transparency = 0
		}):Play()
		task.wait(0.85)
	end

	task.spawn(function()
		while element and element.Parent do
			pulse()
		end
	end)
end

-- ================================================
-- BUTTON HOVER + CLICK (scale feedback)
-- ================================================
function TweenWrapper:ButtonHover(btn)
	if not btn or not btn:IsA("TextButton") then return end

	-- Avoid double-wiring
	if btn:GetAttribute("_hoverWired") then return end
	btn:SetAttribute("_hoverWired", true)

	local origSize = btn.Size
	local stroke   = btn:FindFirstChildOfClass("UIStroke")

	btn.MouseEnter:Connect(function()
		TweenService:Create(btn, makeTI(T_SHORT), {
			Size = UDim2.new(
				origSize.X.Scale * 1.05, origSize.X.Offset,
				origSize.Y.Scale * 1.05, origSize.Y.Offset
			),
			BackgroundTransparency = math.max(0, (btn.BackgroundTransparency or 0) - 0.1)
		}):Play()
		if stroke then
			TweenService:Create(stroke, makeTI(T_SHORT), { Color = NEON, Transparency = 0 }):Play()
		end
		TweenWrapper:PlaySound("hover")
	end)

	btn.MouseLeave:Connect(function()
		TweenService:Create(btn, makeTI(T_SHORT), {
			Size = origSize,
			BackgroundTransparency = btn:GetAttribute("_origTransp") or 0.3
		}):Play()
		if stroke then
			TweenService:Create(stroke, makeTI(T_SHORT), {
				Color = Color3.fromRGB(50, 50, 65),
				Transparency = 0.5
			}):Play()
		end
	end)

	btn.MouseButton1Down:Connect(function()
		TweenService:Create(btn, makeTI(0.1), {
			Size = UDim2.new(
				origSize.X.Scale * 0.95, origSize.X.Offset,
				origSize.Y.Scale * 0.95, origSize.Y.Offset
			)
		}):Play()
	end)

	btn.MouseButton1Up:Connect(function()
		TweenService:Create(btn, makeTI(T_SHORT, Enum.EasingStyle.Back), {
			Size = origSize
		}):Play()
		TweenWrapper:PlaySound("click")
	end)
end

-- ================================================
-- APPLY HOVER TO ALL BUTTONS IN A CONTAINER
-- ================================================
function TweenWrapper:ApplyHoverToAll(container)
	if not container then return end
	for _, desc in ipairs(container:GetDescendants()) do
		if desc:IsA("TextButton") then
			self:ButtonHover(desc)
		end
	end
	container.DescendantAdded:Connect(function(desc)
		if desc:IsA("TextButton") then
			self:ButtonHover(desc)
		end
	end)
end

-- ================================================
-- PAGE TRANSITION (fade + slight upward slide)
-- ================================================
function TweenWrapper:PageTransitionIn(frame, duration)
	if not frame then return end
	duration = duration or T_MED
	frame.Visible = true

	if frame:IsA("CanvasGroup") then
		frame.GroupTransparency = 1
		TweenService:Create(frame, makeTI(duration), { GroupTransparency = 0 }):Play()
	end

	-- Slight upward slide (20px)
	local orig = frame.Position
	local startY = UDim2.new(orig.X.Scale, orig.X.Offset, orig.Y.Scale, orig.Y.Offset + 20)
	frame.Position = startY
	TweenService:Create(frame, makeTI(duration), { Position = orig }):Play()

	TweenWrapper:PlaySound("whoosh")
end

function TweenWrapper:PageTransitionOut(frame, duration)
	if not frame then return end
	duration = duration or T_SHORT
	if frame:IsA("CanvasGroup") then
		local t = TweenService:Create(frame, makeTI(duration), { GroupTransparency = 1 })
		t.Completed:Connect(function() frame.Visible = false end)
		t:Play()
	else
		frame.Visible = false
	end
end

-- ================================================
-- BAR RESIZE (finance / stat bars)
-- ================================================
function TweenWrapper:ResizeBar(fill, pct, duration)
	if not fill then return end
	pct = math.clamp(pct, 0, 1)
	TweenService:Create(fill, makeTI(duration or T_LONG), {
		Size = UDim2.new(pct, 0, 1, 0)
	}):Play()
end

-- ================================================
-- SOUND (built-in Roblox SFX assets)
-- Volumes kept low (0.2–0.3) per spec
-- ================================================
local SFX_IDS = {
	hover  = 6026984224,   -- soft tick
	click  = 6026984224,   -- subtle click (same family, short)
	whoosh = 5982888365,   -- whoosh
}
local _sounds = {}

local function getSound(sfxName)
	if _sounds[sfxName] then return _sounds[sfxName] end
	local id = SFX_IDS[sfxName]
	if not id then return nil end
	local s = Instance.new("Sound")
	s.SoundId  = "rbxassetid://" .. id
	s.Volume   = (sfxName == "whoosh") and 0.2 or 0.15
	s.RollOffMaxDistance = 0
	s.Parent   = SoundService
	_sounds[sfxName] = s
	return s
end

function TweenWrapper:PlaySound(sfxName)
	local s = getSound(sfxName)
	if s then
		s:Play()
	end
end

return TweenWrapper

_______________________________________________

local TweenService = game:GetService("TweenService")
local PageManager = {}

local Pages = {}
local CurrentPage = nil
local CurrentMode = "PLAYER" -- Default mode badge

-- Page title map
local PAGE_TITLES = {
	MainMenu   = "MAIN MENU",
	Dashboard  = "CAREER MODE",
	Training   = "TRAINING",
	Stats      = "PLAYER STATS",
	Transfers  = "TRANSFER MARKET",
	MatchSim   = "MATCH SIMULATION",
	Settings   = "SETTINGS"
}

function PageManager:SetMode(mode)
	CurrentMode = mode
	-- Update all page title bars
	for _, page in pairs(Pages) do
		local bar = page:FindFirstChild("PageTitleBar")
		if bar then
			local badge = bar:FindFirstChild("ModeBadge")
			if badge then
				badge.Text = mode
				badge.BackgroundColor3 = (mode == "MANAGER")
					and Color3.fromRGB(0, 120, 220)
					or  Color3.fromRGB(0, 180, 100)
			end
		end
	end
end

function PageManager:RegisterPage(name, frame)
	Pages[name] = frame
	frame.Visible = false
	if frame:IsA("CanvasGroup") then
		frame.GroupTransparency = 1
	else
		frame.BackgroundTransparency = 1
	end
	print("[UI DEBUG] Page '" .. name .. "' successfully registered.")
end

function PageManager:SwitchPage(targetName, UIController)
	local target = Pages[targetName]
	if not target then
		warn("[PageManager] Page not found: " .. targetName)
		return
	end

	print("[UI DEBUG] Switching Page to: " .. targetName)

	-- Update title bar
	local titleBar = target:FindFirstChild("PageTitleBar")
	if titleBar then
		local titleLbl = titleBar:FindFirstChild("TitleLabel")
		if titleLbl then
			titleLbl.Text = PAGE_TITLES[targetName] or targetName:upper()
		end
		local badge = titleBar:FindFirstChild("ModeBadge")
		if badge then
			badge.Text = CurrentMode
			badge.BackgroundColor3 = (CurrentMode == "MANAGER")
				and Color3.fromRGB(0, 120, 220)
				or  Color3.fromRGB(0, 180, 100)
		end
	end

	-- Fade out current
	if CurrentPage and CurrentPage ~= target then
		local prev = CurrentPage
		if prev:IsA("CanvasGroup") then
			TweenService:Create(prev, TweenInfo.new(0.25), {GroupTransparency = 1}):Play()
		end
		task.delay(0.25, function()
			prev.Visible = false
		end)
	end

	-- Fade in target
	target.Visible = true
	local origPos = target.Position
	if target:IsA("CanvasGroup") then
		target.GroupTransparency = 1
		target.Position = UDim2.new(origPos.X.Scale, origPos.X.Offset, origPos.Y.Scale, origPos.Y.Offset + 30) -- Start lower

		TweenService:Create(target, TweenInfo.new(0.35, Enum.EasingStyle.Cubic, Enum.EasingDirection.Out), {
			GroupTransparency = 0,
			Position = origPos
		}):Play()
	else
		target.BackgroundTransparency = 1
		target.Position = UDim2.new(origPos.X.Scale, origPos.X.Offset, origPos.Y.Scale, origPos.Y.Offset + 30)
		TweenService:Create(target, TweenInfo.new(0.35, Enum.EasingStyle.Cubic, Enum.EasingDirection.Out), {
			BackgroundTransparency = 0,
			Position = origPos
		}):Play()
	end

	CurrentPage = target
end

return PageManager

______________________________________________

TutorialManager

-- ================================================
-- TUTORIAL CONTROLLER (Client/Core) v1.0
-- ================================================
-- Controls the 5-step Manager Mode onboarding flow.
-- Smooth camera zooms, dark overlays, and step-by-step guidance.
-- ================================================

local TweenService = game:GetService("TweenService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Core = script.Parent
local TweenWrapper = require(Core:WaitForChild("TweenServiceWrapper"))

local TutorialController = {}

local TutorialOverlay = nil
local CurrentStep = 0
local TutorialData = {
	{
		Title = "WELCOME TO MANAGER MODE",
		Text = "Your journey into football management begins here. Let's get you settled.",
		Highlight = nil -- Whole screen
	},
	{
		Title = "THE WORLD MAP",
		Text = "This is your global hub. Drag to move around and explore different regions.",
		Highlight = "WorldMap"
	},
	{
		Title = "ZOOM & EXPLORE",
		Text = "Click a region to zoom in and see the available leagues and divisions.",
		Highlight = "RegionContainer"
	},
	{
		Title = "LEAGUE SELECTION",
		Text = "Choose your starting division. Lower divisions offer a challenge, while top flights have massive budgets.",
		Highlight = "LeagueList"
	},
	{
		Title = "KICKOFF",
		Text = "Once you've made your choice, confirm to create your club and start your legacy.",
		Highlight = "ConfirmButton"
	}
}

function TutorialController:Init(playerGui)
	local ui = playerGui:WaitForChild("UI", 10)
	if not ui then return end

	TutorialOverlay = ui:FindFirstChild("TutorialOverlay")
	if not TutorialOverlay then
		-- Create the overlay if it doesn't exist
		TutorialOverlay = Instance.new("CanvasGroup")
		TutorialOverlay.Name = "TutorialOverlay"
		TutorialOverlay.Size = UDim2.new(1, 0, 1, 0)
		TutorialOverlay.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
		TutorialOverlay.BackgroundTransparency = 0.4
		TutorialOverlay.Visible = false
		TutorialOverlay.Parent = ui

		local content = Instance.new("Frame")
		content.Name = "Content"
		content.Size = UDim2.new(0, 400, 0, 200)
		content.Position = UDim2.new(0.5, -200, 0.5, -100)
		content.BackgroundColor3 = Color3.fromRGB(25, 25, 35)
		content.Parent = TutorialOverlay
		Instance.new("UICorner", content).CornerRadius = UDim.new(0, 12)
		local stroke = Instance.new("UIStroke", content)
		stroke.Color = Color3.fromRGB(0, 255, 150)
		stroke.Thickness = 2

		local title = Instance.new("TextLabel")
		title.Name = "Title"
		title.Size = UDim2.new(1, -20, 0, 40)
		title.Position = UDim2.new(0, 10, 0, 10)
		title.BackgroundTransparency = 1
		title.TextColor3 = Color3.fromRGB(0, 255, 150)
		title.Font = Enum.Font.GothamBold
		title.TextSize = 20
		title.Text = "TITLE"
		title.Parent = content

		local desc = Instance.new("TextLabel")
		desc.Name = "Description"
		desc.Size = UDim2.new(1, -20, 1, -100)
		desc.Position = UDim2.new(0, 10, 0, 60)
		desc.BackgroundTransparency = 1
		desc.TextColor3 = Color3.new(1, 1, 1)
		desc.Font = Enum.Font.Gotham
		desc.TextSize = 14
		desc.TextWrapped = true
		desc.Text = "Description goes here..."
		desc.Parent = content

		local btn = Instance.new("TextButton")
		btn.Name = "NextBtn"
		btn.Size = UDim2.new(0, 120, 0, 35)
		btn.Position = UDim2.new(0.5, -60, 1, -45)
		btn.BackgroundColor3 = Color3.fromRGB(0, 255, 150)
		btn.TextColor3 = Color3.fromRGB(15, 15, 20)
		btn.Font = Enum.Font.GothamBold
		btn.TextSize = 14
		btn.Text = "NEXT"
		btn.Parent = content
		Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 8)

		btn.MouseButton1Click:Connect(function()
			self:Next()
		end)
	end
end

function TutorialController:Start()
	CurrentStep = 0
	self:Next()
end

function TutorialController:Next()
	CurrentStep += 1
	if CurrentStep > #TutorialData then
		self:End()
		return
	end

	local data = TutorialData[CurrentStep]
	local content = TutorialOverlay:FindFirstChild("Content")
	local title = content:FindFirstChild("Title")
	local desc = content:FindFirstChild("Description")
	local nextBtn = content:FindFirstChild("NextBtn")

	title.Text = data.Title
	desc.Text = data.Text
	nextBtn.Text = (CurrentStep == #TutorialData) and "FINISH" or "NEXT"

	if not TutorialOverlay.Visible then
		TweenWrapper:FadeIn(TutorialOverlay, 0.4)
	end

	-- Handle highlight and animations
	if data.Highlight == "RegionContainer" then
		-- Logic to highlight specific map areas or simulate zoom could go here
		TweenWrapper:PlaySound("whoosh")
	end
end

function TutorialController:End()
	TweenWrapper:FadeOut(TutorialOverlay, 0.4)
	-- Save tutorial completion state
	local Remotes = ReplicatedStorage:WaitForChild("Remotes")
	local UpdateData = Remotes:WaitForChild("UpdateData")
	UpdateData:FireServer("SetTutorialDone", true)
end

return TutorialController


_____________________________________________

Admin Panel

-- ================================================
-- ADMIN PANEL (StarterGui.UI.Core - ModuleScript)
-- ================================================
-- Premium AAA Admin interface for Manager Mode.
-- Accessible only by OWNER/ADMIN roles.
-- ================================================
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TweenService      = game:GetService("TweenService")
local Players           = game:GetService("Players")

local Remotes           = ReplicatedStorage:WaitForChild("Remotes")
local AdminFunc         = Remotes:WaitForChild("AdminFunc")
local AdminEvent        = Remotes:WaitForChild("AdminEvent")

local AdminPanel = {}

-- Styles
local NEON  = Color3.fromRGB(0, 255, 150)
local DARK  = Color3.fromRGB(15, 15, 20)
local RED   = Color3.fromRGB(255, 50, 50)
local TEXT  = Color3.fromRGB(230, 230, 240)

function AdminPanel:Init(parentUI, role)
	if not role or (role ~= "OWNER" and role ~= "ADMIN") then return end

	local panel = Instance.new("Frame")
	panel.Name = "AdminPanel"
	panel.Size = UDim2.new(0, 600, 0, 450)
	panel.Position = UDim2.new(0.5, -300, 0.5, -225)
	panel.BackgroundColor3 = DARK
	panel.BorderSizePixel = 0
	panel.Visible = false
	panel.Parent = parentUI

	local corner = Instance.new("UICorner", panel)
	local stroke = Instance.new("UIStroke", panel)
	stroke.Color = NEON
	stroke.Thickness = 1.5

	-- Title Bar
	local title = Instance.new("TextLabel", panel)
	title.Size = UDim2.new(1, -20, 0, 40)
	title.Position = UDim2.new(0, 10, 0, 5)
	title.Text = "🛡️ ADMIN COMMAND CENTER [" .. role .. "]"
	title.TextColor3 = NEON
	title.Font = Enum.Font.Oswald
	title.TextSize = 20
	title.TextXAlignment = Enum.TextXAlignment.Left
	title.BackgroundTransparency = 1

	-- Logging Area
	local logFrame = Instance.new("ScrollingFrame", panel)
	logFrame.Size = UDim2.new(1, -20, 0, 150)
	logFrame.Position = UDim2.new(0, 10, 0, 50)
	logFrame.BackgroundColor3 = Color3.fromRGB(10, 10, 15)
	logFrame.BorderSizePixel = 0
	logFrame.CanvasSize = UDim2.new(0,0,0,1000)

	local listLayout = Instance.new("UIListLayout", logFrame)
	listLayout.Padding = UDim.new(0, 4)

	local function addLog(entry)
		local lbl = Instance.new("TextLabel", logFrame)
		lbl.Size = UDim2.new(1, -5, 0, 20)
		lbl.Text = " [" .. os.date("%H:%M", entry.Timestamp) .. "] " .. entry.Admin .. ": " .. entry.Action .. " -> " .. entry.Details
		lbl.TextColor3 = TEXT
		lbl.BackgroundTransparency = 1
		lbl.TextSize = 14
		lbl.Font = Enum.Font.RobotoMono
		lbl.TextXAlignment = Enum.TextXAlignment.Left
	end

	-- Commands Area
	local cmdFrame = Instance.new("Frame", panel)
	cmdFrame.Size = UDim2.new(1, -20, 0, 220)
	cmdFrame.Position = UDim2.new(0, 10, 0, 210)
	cmdFrame.BackgroundTransparency = 1

	local function createBtn(name, pos, color, callback)
		local btn = Instance.new("TextButton", cmdFrame)
		btn.Size = UDim2.new(0, 180, 0, 35)
		btn.Position = pos
		btn.Text = name
		btn.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
		btn.TextColor3 = color
		btn.Font = Enum.Font.GothamMedium
		btn.TextSize = 14

		local bC = Instance.new("UICorner", btn)
		local bS = Instance.new("UIStroke", btn)
		bS.Color = color
		bS.Thickness = 0.5

		btn.MouseButton1Click:Connect(callback)
		return btn
	end

	-- 1. Modify Money
	createBtn("💸 SET BUDGET", UDim2.new(0,0,0,0), NEON, function()
		local username = parentUI:FindFirstChild("AdminUserTarget") and parentUI.AdminUserTarget.Text or Players.LocalPlayer.Name
		local amount   = parentUI:FindFirstChild("AdminAmountInput") and tonumber(parentUI.AdminAmountInput.Text) or 10000000
		AdminFunc:InvokeServer("SetMoney", { Username = username, Amount = amount })
	end)

	-- 2. Force Result
	createBtn("⚽ FORCE WIN", UDim2.new(0,200,0,0), NEON, function()
		AdminFunc:InvokeServer("ForceResult", { Result = "WIN" })
	end)

	-- 3. World Reset (Owner Only)
	if role == "OWNER" then
		createBtn("🔁 WORLD RESET", UDim2.new(0,400,0,0), RED, function()
			AdminFunc:InvokeServer("WorldReset")
		end)
	end

	-- Listen for live logs
	AdminEvent.OnClientEvent:Connect(function(type, data)
		if type == "NewLog" then addLog(data) end
	end)

	-- Initial log fetch
	local logs = AdminFunc:InvokeServer("GetLogs")
	if logs then for _, l in ipairs(logs) do addLog(l) end end

	AdminPanel.Instance = panel
end

function AdminPanel:Toggle()
	if not self.Instance then return end
	self.Instance.Visible = not self.Instance.Visible
end

return AdminPanel

_____________________________________________

MatchSimClient

local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TweenService = game:GetService("TweenService")
local MatchUpdate = ReplicatedStorage:WaitForChild("MatchUpdate")

local UI = script.Parent.Parent
local MatchSimPage = UI.Pages:WaitForChild("MatchSim")
local Pitch = MatchSimPage:WaitForChild("Pitch")
local HUD = MatchSimPage:WaitForChild("HUD")

-- Map server player names to UI containers
local HOME_NAMES = {"Valdez","Santos","Ramos","Cruz","Diaz","Vargas","Moreno","Silva","Reyes","Gomez","Torres"}
local AWAY_NAMES = {"Miller","Jones","Brown","Taylor","Wilson","Davis","Harris","Martin","Garcia","Lewis","Walker"}

local TWEEN_INFO = TweenInfo.new(0.08, Enum.EasingStyle.Linear)
local BALL_TWEEN = TweenInfo.new(0.05, Enum.EasingStyle.Linear)

MatchUpdate.OnClientEvent:Connect(function(data, eventType, result)
	-- Update HUD
	local scoreboard = HUD:FindFirstChild("Scoreboard")
	if scoreboard then scoreboard.Text = data.Score.Home .. " - " .. data.Score.Away end

	local timerLbl = HUD:FindFirstChild("TimerLabel")
	if timerLbl then
		local minute = math.floor(data.GameMinute or 0)
		if data.IsExtraTime then
			timerLbl.Text = "90+" .. (minute - 90) .. "'"
		else
			timerLbl.Text = minute .. "'"
		end
	end

	-- Smoothly move Home players
	for i, p in ipairs(data.Players.Home) do
		local name = HOME_NAMES[i]
		local container = Pitch:FindFirstChild("P_" .. name)
		if container then
			TweenService:Create(container, TWEEN_INFO, {
				Position = UDim2.new(p.Pos.X, 0, p.Pos.Y, 0)
			}):Play()
		end
	end

	-- Smoothly move Away players
	for i, p in ipairs(data.Players.Away) do
		local name = AWAY_NAMES[i]
		local container = Pitch:FindFirstChild("P_" .. name)
		if container then
			TweenService:Create(container, TWEEN_INFO, {
				Position = UDim2.new(p.Pos.X, 0, p.Pos.Y, 0)
			}):Play()
		end
	end

	-- Smoothly move Ball
	local ball = Pitch:FindFirstChild("Ball")
	if ball then
		TweenService:Create(ball, BALL_TWEEN, {
			Position = UDim2.new(data.Ball.X, 0, data.Ball.Y, 0)
		}):Play()
	end

	-- Match End handling
	if eventType == "MatchEnd" then
		local kickOffBtn = MatchSimPage:FindFirstChild("KickOffButton")
		if kickOffBtn then
			kickOffBtn.Text = "⚽  KICK OFF"
			kickOffBtn.BackgroundColor3 = Color3.fromRGB(0, 255, 154)
			kickOffBtn.TextColor3 = Color3.fromRGB(10, 10, 15)
		end
		print("🏆 FULL TIME! " .. (result or ""))
	end
end)


_____________________________________________

ServerScritpService

WorldStateService

-- ================================================
-- WORLD STATE SERVICE (ModuleScript) - v3.0 FINAL
-- ================================================
-- Central data hub for ALL clubs, players, and world state.
-- Every system reads and writes through this module.
-- ================================================
local WorldStateService = {}

WorldStateService.Universe = {
	Clubs    = {},
	Players  = {},
	Fixtures = {}, -- [Region][Division][Week] = { {Home, Away}, ... }
	FreeAgents = {},
	TransferHistory = {},
	NewsFeed = {},
	GlobalEvents = {},
	Season   = 2025,
	Week     = 1,
}

local SEASON_DURATION = 38 -- Weeks per season

-- ================================================
-- 📅 FIXTURES (Round Robin Algorithm)
-- ================================================
function WorldStateService:GenerateFixturesForDivision(region, div)
	local clubs = {}
	for id, c in pairs(self.Universe.Clubs) do
		if c.Region == region and c.Division == div then
			table.insert(clubs, id)
		end
	end

	if #clubs % 2 ~= 0 then table.insert(clubs, "BYE") end -- Pad for even number
	local numClubs = #clubs
	local numWeeks = (numClubs - 1) * 2 -- Home and Away

	if not self.Universe.Fixtures[region] then self.Universe.Fixtures[region] = {} end
	self.Universe.Fixtures[region][div] = {}

	for w = 1, numWeeks do
		self.Universe.Fixtures[region][div][w] = {}
		for i = 1, numClubs / 2 do
			local hIdx = i
			local aIdx = numClubs - i + 1
			local h, a = clubs[hIdx], clubs[aIdx]

			if h ~= "BYE" and a ~= "BYE" then
				-- Alternate home/away in second half of season
				if w > (numWeeks / 2) then
					table.insert(self.Universe.Fixtures[region][div][w], { Home = a, Away = h })
				else
					table.insert(self.Universe.Fixtures[region][div][w], { Home = h, Away = a })
				end
			end
		end
		-- Rotate clubs (keep first one fixed)
		table.insert(clubs, 2, table.remove(clubs))
	end
end

-- ================================================
-- 🏟️ CLUB TEMPLATE
-- ================================================
local ClubTemplate = {
	ID = "", Name = "", ShortName = "FC", Region = "", Division = 10,
	Motto = "Victory", ManagerName = "Manager",
	Kits = { Home = {R=0,G=100,B=220}, Away = {R=255,G=255,B=255}, Third = {R=0,G=0,B=0} },
	Logo = { Shape = "Shield", Icon = "⚽", Color = {R=0,G=100,B=220} },
	Reputation = 100, Philosophy = "Balanced", Playstyle = "Possession", Form = 0,
	Budget = 1000000, WageBill = 0, MatchIncome = 0, SponsorIncome = 0,
	Facilities = { Training = 1, Youth = 1, Recovery = 1 },
	Roster = {}, YouthAcademy = {}, ScoutMissions = {},
	Sponsors = {},
	Formation = "4-3-3", StartingXI = {},
	LeagueStats = { Played=0, Wins=0, Draws=0, Losses=0, Points=0, GF=0, GA=0, GD=0 },
	RecentResults = {}, -- Last 5: "W", "D", "L"
	BoardConfidence = 50,
}

-- ================================================
-- 🧍 PLAYER TEMPLATE
-- ================================================
local PlayerTemplate = {
	ID = "", Name = "", Age = 20, Position = "CM", Nationality = "ENG",
	NationalityFlag = "🏴󠁧󠁢󠁥󠁮󠁧󠁿",
	OVR = 60, Potential = 75, Form = 0, Morale = 80, Condition = 100,
	Value = 500000, Wage = 2000,
	Personality = "Balanced", GrowthRate = 1.0,
	Club = "Free Agent", IsYouth = false,
	Stats = { Shooting=60, Passing=60, Dribbling=60, Defending=60, Pace=60, Physical=60 },
}

-- ================================================
-- NATIONALITY DATA
-- ================================================
local NATIONALITY_DATA = {
	England      = { flag = "🏴", code = "ENG" },
	Spain        = { flag = "🇪🇸", code = "ESP" },
	Germany      = { flag = "🇩🇪", code = "DEU" },
	France       = { flag = "🇫🇷", code = "FRA" },
	Italy        = { flag = "🇮🇹", code = "ITA" },
	Brazil       = { flag = "🇧🇷", code = "BRA" },
	Argentina    = { flag = "🇦🇷", code = "ARG" },
	Portugal     = { flag = "🇵🇹", code = "POR" },
	Japan        = { flag = "🇯🇵", code = "JPN" },
	Korea        = { flag = "🇰🇷", code = "KOR" },
	["Saudi Arabia"] = { flag = "🇸🇦", code = "SAU" },
	Nigeria      = { flag = "🇳🇬", code = "NGA" },
	Russia       = { flag = "🇷🇺", code = "RUS" },
	Poland       = { flag = "🇵🇱", code = "POL" }
}
local NAT_LIST = {"England","Spain","Germany","France","Italy","Brazil","Argentina","Portugal","Japan","Korea","Saudi Arabia","Nigeria","Russia","Poland"}

-- ================================================
-- 🧬 THE MEGA NAME POOLS
-- ================================================
local FIRST_NAMES = {
	WESTERN = {"Liam", "Noah", "Ethan", "Mason", "Lucas", "Oliver", "James", "Aiden", "Logan", "Elijah", "Benjamin", "Henry", "Alexander", "Sebastian", "William", "Daniel", "Samuel", "David", "Michael", "Thomas", "Joseph", "Matthew", "Andrew", "Christopher", "Nathan", "Gabriel"},
	LATIN   = {"Mateo", "Diego", "Carlos", "Javier", "Marco", "Antonio", "Rafael", "Sergio", "Pedro", "Luis", "Andres", "Miguel", "Ricardo", "Eduardo", "Felipe", "Santiago", "Jorge", "Hector", "Alonso", "Raul", "Pablo", "Victor", "Adrian", "Manuel", "Tomas", "Ignacio", "Emilio"},
	ASIA    = {"Kenji", "Hiro", "Sota", "Yuto", "Haruki", "Ren", "Daichi", "Kaito", "Minho", "Joon", "Haru", "Riku", "Sho", "Taiki", "Yuma", "Jin", "Seojun", "Hyun", "Woojin", "Jisoo", "Takumi", "Ryo", "Shun", "Yuki", "Akira", "Kenta", "Daigo", "Naoki", "Taro", "Issei"},
	ARABIC  = {"Omar", "Amir", "Karim", "Zayd", "Hassan", "Idris", "Malik", "Tarek", "Samir", "Nabil", "Youssef", "Ali", "Ahmed", "Mohamed", "Ibrahim", "Rashid", "Faris", "Khalid", "Hamza", "Said", "Zakaria", "Musa", "Adil", "Ismail", "Rami", "Anwar", "Bilal", "Yasin", "Nadim"},
	SLAVIC  = {"Ivan", "Dmitri", "Viktor", "Aleks", "Nikolai", "Sergei", "Luka", "Milan", "Petar", "Dario", "Bogdan", "Andrei", "Stefan", "Mikhail", "Georgi", "Vlad", "Boris", "Luka", "Emilian", "Radek", "Tomasz", "Kacper", "Dominik", "Pavel", "Jovan", "Nenad", "Zoran", "Marko"}
}

local LAST_NAMES = {
	ENGLISH = {"Smith", "Johnson", "Brown", "Williams", "Jones", "Miller", "Davis", "Wilson", "Moore", "Taylor", "Anderson", "Thomas", "Jackson", "White", "Harris", "Martin", "Thompson", "Garcia", "Martinez", "Robinson", "Clark", "Rodriguez", "Lewis", "Lee", "Walker", "Hall"},
	IBERIAN = {"Silva", "Santos", "Costa", "Pereira", "Oliveira", "Lima", "Almeida", "Rocha", "Carvalho", "Ribeiro", "Mendes", "Barbosa", "Gomes", "Duarte", "Freitas", "Cardoso", "Nunes", "Pires", "Fernandez", "Lopez", "Gonzalez", "Hernandez", "Martinez", "Alvarez", "Romero", "Torres"},
	ITALIAN = {"Rossi", "Romano", "Ferrari", "Bianchi", "Ricci", "Conti", "Esposito", "Moretti", "Colombo", "Gallo", "De Luca", "Marino", "Greco", "Bruno", "Rizzo", "De Santis", "Lombardi", "Caruso"},
	SLAVIC  = {"Ivanov", "Petrov", "Volkov", "Smirnov", "Kuznetsov", "Popov", "Sokolov", "Morozov", "Fedorov", "Orlov", "Makarov", "Lebedev", "Novikov", "Pavlov", "Stepanov", "Zakharov"},
	ASIAN   = {"Nakamura", "Tanaka", "Sato", "Suzuki", "Yamamoto", "Kobayashi", "Kato", "Yoshida", "Kim", "Park", "Choi", "Jung", "Lim", "Kang", "Yoon", "Han", "Seo", "Lee", "Hwang"},
	AFRICAN = {"Haddad", "Farah", "Ali", "Hassan", "Mansour", "Rahman", "Ibrahim", "Said", "Karim", "Nasser", "Abdullah", "Omar", "Zaki", "Khalil", "Yusuf", "Mahdi", "Saeed", "Ismail"}
}

local CLUB_PREFIXES = {"VL", "RB", "ST", "NC", "AV", "PE", "LY", "MV", "LC", "BN", "LS", "MI", "AM", "NP", "DU", "MN", "BR", "RW", "HC", "SL", "IR", "SF", "ET", "DH", "WH", "BH", "IH", "SC", "RC", "OS", "TP", "TR", "SE", "RA", "CT", "DB", "HT", "MX", "DK", "GV", "FR", "BL", "CR", "GR", "VL", "ZR", "XR", "QN", "JT", "KP", "OR", "VT", "NX", "ZN", "UL", "AX", "PY", "SK", "DR", "FW", "HB", "TW", "GL", "PR", "KN", "LD", "SV", "RT", "CM", "FK"}
local CLUB_ROOTS    = {"Royal", "Valen", "North", "South", "East", "West", "Iron", "Storm", "Shadow", "Nova", "Phoenix", "Titan", "Apex", "Crest", "Harbor", "Ridge", "Valley", "Forge", "Drift", "Crystal", "Golden", "Silver", "Black", "Red", "Blue", "White", "Crimson", "Obsidian", "Lunar", "Solar", "Stormborn", "Ironvale", "Frost", "Ember", "Thunder", "Midnight", "Dawn", "Eclipse", "Inferno", "Vortex", "Quantum", "Zenith", "Vertex", "Orbit", "Galaxy", "Meteor", "Atlas", "NovaCore", "Stone", "Cedar", "Redwood", "Ashen", "Granite", "Steel", "Alloy", "Blade", "Arrow", "Falcon", "Wolf", "Lion", "Eagle", "Panther", "Dragon", "Serpent", "Hydra", "Phoenixborn", "Sky", "Ocean"}
local CLUB_SUFFIXES = {"FC", "SC", "AC", "Real", "United", "City", "Athletic", "Club", "Dynamo", "Rangers", "Wanderers", "Rovers", "Kings", "Warriors", "Legends", "Elite", "Prime", "Nova", "Iron", "Royal", "Imperial", "Continental", "National", "Global", "Supreme", "United FC", "Association", "Brotherhood", "Legion", "Syndicate", "Order", "Federation", "Academy", "Division", "Select", "Pro", "Mega", "Ultra", "Hyper"}

-- ================================================
-- ECONOMY SCALING BY DIVISION (v3 — Spec-accurate)
-- ================================================
--[[ Budget and Income scale:
    Div 10 → £500K   / £20K/wk
    Div 9  → £1M     / £35K/wk
    Div 8  → £2M     / £60K/wk
    Div 7  → £4M     / £100K/wk
    Div 6  → £8M     / £150K/wk
    Div 5  → £15M    / £250K/wk
    Div 4  → £30M    / £400K/wk
    Div 3  → £60M    / £650K/wk
    Div 2  → £120M   / £900K/wk
    Div 1  → £250M   / £2M/wk
]]
local DIVISION_ECONOMY = {
	[1]  = { budget = 250000000, matchIncome = 2000000 },
	[2]  = { budget = 120000000, matchIncome = 900000  },
	[3]  = { budget = 60000000,  matchIncome = 650000  },
	[4]  = { budget = 30000000,  matchIncome = 400000  },
	[5]  = { budget = 15000000,  matchIncome = 250000  },
	[6]  = { budget = 8000000,   matchIncome = 150000  },
	[7]  = { budget = 4000000,   matchIncome = 100000  },
	[8]  = { budget = 2000000,   matchIncome = 60000   },
	[9]  = { budget = 1000000,   matchIncome = 35000   },
	[10] = { budget = 500000,    matchIncome = 20000   },
}

-- Sponsor tiers per division
local SPONSOR_TIERS = {
	[1]  = 5000000, [2] = 2000000, [3] = 1000000, [4] = 500000,
	[5]  = 250000,  [6] = 120000,  [7] = 60000,   [8] = 30000,
	[9]  = 15000,   [10]= 5000
}

local SPONSOR_BRANDS = {
	{ Name = "Volta Energy", 	Type = "Performance" },
	{ Name = "AeroFly", 		Type = "WinStreak" },
	{ Name = "PrimeBank", 		Type = "Finance" },
	{ Name = "Cortex Cyber", 	Type = "Youth" },
	{ Name = "Neon Motors", 	Type = "Reputation" }
}

-- ================================================
-- WAGE FORMULA (OVR-based, per spec)
-- ================================================
function WorldStateService:CalcWage(ovr)
	if ovr >= 95 then return 400000
	elseif ovr >= 90 then return 200000
	elseif ovr >= 85 then return 100000
	elseif ovr >= 80 then return 50000
	elseif ovr >= 75 then return 25000
	elseif ovr >= 70 then return 10000
	elseif ovr >= 65 then return 5000
	else return 2000 end
end

-- ================================================
-- 🛠️ CLUB CREATION
-- ================================================
function WorldStateService:CreateClub(id, name, region, division, budget, playstyle)
	local eco = DIVISION_ECONOMY[division] or DIVISION_ECONOMY[10]
	local newClub = {}
	for k, v in pairs(ClubTemplate) do newClub[k] = v end
	-- Deep copy nested tables
	newClub.Kits      = { Home={R=0,G=100,B=220}, Away={R=255,G=255,B=255}, Third={R=0,G=0,B=0} }
	newClub.Logo      = { Shape="Shield", Icon="⚽", Color={R=0,G=100,B=220} }
	newClub.Facilities= { Training=1, Youth=1, Recovery=1 }
	newClub.Roster    = {} newClub.YouthAcademy = {} newClub.ScoutMissions = {}
	newClub.Sponsors  = {} newClub.StartingXI   = {} newClub.RecentResults = {}
	newClub.LeagueStats = { Played=0, Wins=0, Draws=0, Losses=0, Points=0, GF=0, GA=0 }

	newClub.ID        = id
	newClub.Name      = name
	newClub.Region    = region
	newClub.Division  = division
	newClub.Budget    = budget or eco.budget
	newClub.Playstyle = playstyle or "Balanced"
	newClub.MatchIncome   = eco.matchIncome
	newClub.SponsorIncome = SPONSOR_TIERS[division] or 5000
	newClub.OVR       = 50

	-- Generate 3 Sponsors
	for i = 1, 3 do
		local brand = SPONSOR_BRANDS[math.random(#SPONSOR_BRANDS)]
		table.insert(newClub.Sponsors, {
			Name = brand.Name,
			Type = brand.Type,
			GoalAmount = 5, -- e.g. 5 wins
			Progress   = 0,
			Reward     = SPONSOR_TIERS[division] or 10000,
			Claimed    = false
		})
	end

	self.Universe.Clubs[id] = newClub
	return newClub
end

local function randomizeStats(ovr, pos)
	local s = {Shooting=ovr, Passing=ovr, Dribbling=ovr, Defending=ovr, Pace=ovr, Physical=ovr}
	local noise = 15

	-- Position-based weighting
	if pos == "ST" or pos == "LW" or pos == "RW" then
		s.Shooting += 8 s.Pace += 5 s.Defending -= 10
	elseif pos == "CM" or pos == "CAM" or pos == "CDM" then
		s.Passing += 8 s.Dribbling += 5 s.Defending -= 2
	elseif pos == "CB" or pos == "LB" or pos == "RB" then
		s.Defending += 10 s.Physical += 5 s.Shooting -= 12
	elseif pos == "GK" then
		s.Defending += 15 s.Passing -= 10 s.Shooting -= 20
	end

	for k, v in pairs(s) do 
		s[k] = math.clamp(v + math.random(-noise, noise), 1, 99) 
	end
	return s
end

-- ================================================
-- 🧍 PLAYER CREATION
-- ================================================
function WorldStateService:CreatePlayer(id, name, age, position, ovr, pot, personality)
	local nat = NAT_LIST[math.random(#NAT_LIST)]
	local natData = NATIONALITY_DATA[nat]
	local newPlayer = {}
	for k, v in pairs(PlayerTemplate) do newPlayer[k] = v end
	newPlayer.Stats = { Shooting=60, Passing=60, Dribbling=60, Defending=60, Pace=60, Physical=60 }

	newPlayer.ID          = id
	newPlayer.Name        = name
	newPlayer.Age         = age
	newPlayer.Position    = position
	newPlayer.OVR         = ovr
	newPlayer.Potential   = pot
	newPlayer.Personality = personality or "Balanced"
	newPlayer.Nationality = natData.code
	newPlayer.NationalityFlag = natData.flag
	newPlayer.Wage        = math.floor(ovr * 50 * (1 + (pot - ovr) * 0.01))
	newPlayer.Value       = math.max(50000, math.floor((ovr ^ 2) * 500))
	newPlayer.Stats       = randomizeStats(ovr, position)

	self.Universe.Players[id] = newPlayer
	return newPlayer
end

function WorldStateService:GenerateRoster(clubID, region, baseOVR)
	local club = self.Universe.Clubs[clubID]
	if not club then return end

	local positions = {
		"GK", "GK",
		"CB","CB","CB","CB","LB","RB",
		"CM","CM","CM","CDM","CAM",
		"LW","RW","ST","ST","ST"
	}
	local function getPlayerName(nat)
		local poolF = FIRST_NAMES.WESTERN
		local poolL = LAST_NAMES.ENGLISH

		if nat == "ESP" or nat == "ARG" or nat == "BRA" or nat == "POR" then
			poolF = FIRST_NAMES.LATIN
			poolL = LAST_NAMES.IBERIAN
		elseif nat == "ITA" then
			poolF = FIRST_NAMES.LATIN
			poolL = LAST_NAMES.ITALIAN
		elseif nat == "JPN" or nat == "KOR" then
			poolF = FIRST_NAMES.ASIA
			poolL = LAST_NAMES.ASIAN
		elseif nat == "SAU" or nat == "NGA" then
			poolF = FIRST_NAMES.ARABIC
			poolL = LAST_NAMES.AFRICAN
		elseif nat == "RUS" or nat == "POL" then
			poolF = FIRST_NAMES.SLAVIC
			poolL = LAST_NAMES.SLAVIC
		end

		return poolF[math.random(#poolF)] .. " " .. poolL[math.random(#poolL)]
	end

	for i, pos in ipairs(positions) do
		local pID  = clubID .. "_P" .. i
		local pOVR = math.clamp(baseOVR + math.random(-5, 5), 40, 99)
		local pAge = math.random(18, 33)

		local nat  = NAT_LIST[math.random(#NAT_LIST)]
		local natD = NATIONALITY_DATA[nat] or { code = "ENG", flag = "🏴󠁧󠁢󠁥󠁮󠁧󠁿" }

		club.Roster[pID] = {
			ID = pID,
			Name = getPlayerName(natD.code),
			Age = pAge, Position = pos,
			OVR = pOVR, Potential = math.clamp(pOVR + math.random(5, 15), pOVR, 99),
			Morale = 80, Condition = 100, Form = 0,
			Value = math.max(50000, math.floor((pOVR ^ 2) * 500)),
			Wage  = math.floor(pOVR * 50),
			Nationality = natD.code,
			NationalityFlag = natD.flag,
			Club = club.Name,
			Stats = randomizeStats(pOVR, pos)
		}
		club.WageBill += math.floor(pOVR * 50)
	end

	-- Calculate Club OVR
	local total, count = 0, 0
	for _, p in pairs(club.Roster) do total += p.OVR count += 1 end
	if count > 0 then club.OVR = math.floor(total / count) end
end

-- ================================================
-- 📊 STANDINGS
-- ================================================
function WorldStateService:GetDivisionStandings(region, division)
	local t = {}
	for _, c in pairs(self.Universe.Clubs) do
		if c.Region == region and c.Division == division then
			table.insert(t, c)
		end
	end
	table.sort(t, function(a, b)
		local aStats = a.LeagueStats or {Points=0, GD=0, GF=0}
		local bStats = b.LeagueStats or {Points=0, GD=0, GF=0}

		if (aStats.Points or 0) ~= (bStats.Points or 0) then
			return (aStats.Points or 0) > (bStats.Points or 0)
		end
		-- Tie breaker: Goal Difference
		local aGD = (aStats.GD or (aStats.GF-aStats.GA))
		local bGD = (bStats.GD or (bStats.GF-bStats.GA))
		if aGD ~= bGD then return aGD > bGD end
		-- Second tie breaker: OVR (higher tier teams win)
		return (a.OVR or 50) > (b.OVR or 50)
	end)
	return t
end

-- ================================================
-- 🏆 RECORD MATCH RESULT
-- ================================================
function WorldStateService:RecordMatchResult(homeID, awayID, homeScore, awayScore, isSilent)
	local home = self.Universe.Clubs[homeID]
	local away = self.Universe.Clubs[awayID]
	if not home or not away then return end

	home.LeagueStats.Played += 1 away.LeagueStats.Played += 1
	home.LeagueStats.GF += homeScore home.LeagueStats.GA += awayScore
	away.LeagueStats.GF += awayScore away.LeagueStats.GA += homeScore

	-- Calculate GD
	home.LeagueStats.GD = home.LeagueStats.GF - home.LeagueStats.GA
	away.LeagueStats.GD = away.LeagueStats.GF - away.LeagueStats.GA

	home.Budget += home.MatchIncome away.Budget += away.MatchIncome

	-- SPONSOR PAYOUTS (Pay-per-match logic)
	local function paySponsors(club, isWinner)
		local totalEarned = 0
		for _, spon in ipairs(club.Sponsors or {}) do
			if not spon.Claimed then
				local fee = math.floor(spon.Reward / 15) -- Base match fee
				if isWinner then fee = math.floor(fee * 1.5) end -- Win bonus
				club.Budget += fee
				totalEarned += fee
				spon.Progress = math.min(spon.GoalAmount, spon.Progress + 1)
			end
		end
		return totalEarned
	end

	local hEarned = paySponsors(home, homeScore > awayScore)
	local aEarned = paySponsors(away, awayScore > homeScore)

	local function pushResult(club, result)
		table.insert(club.RecentResults, 1, result)
		if #club.RecentResults > 5 then table.remove(club.RecentResults, 6) end
	end

	if homeScore > awayScore then
		home.LeagueStats.Wins += 1   home.LeagueStats.Points += 3
		away.LeagueStats.Losses += 1
		home.Form  = math.clamp(home.Form + 1, -5, 5)
		away.Form  = math.clamp(away.Form - 1, -5, 5)
		home.BoardConfidence = math.clamp((home.BoardConfidence or 50) + 3, 0, 100)
		away.BoardConfidence = math.clamp((away.BoardConfidence or 50) - 2, 0, 100)
		pushResult(home, "W") pushResult(away, "L")
		if not isSilent then self:PushNews("⚽ FT: " .. home.Name .. " " .. homeScore .. "-" .. awayScore .. " " .. away.Name) end
	elseif awayScore > homeScore then
		away.LeagueStats.Wins += 1   away.LeagueStats.Points += 3
		home.LeagueStats.Losses += 1
		away.Form  = math.clamp(away.Form + 1, -5, 5)
		home.Form  = math.clamp(home.Form - 1, -5, 5)
		away.BoardConfidence = math.clamp((away.BoardConfidence or 50) + 3, 0, 100)
		home.BoardConfidence = math.clamp((home.BoardConfidence or 50) - 2, 0, 100)
		pushResult(away, "W") pushResult(home, "L")
		if not isSilent then self:PushNews("⚽ FT: " .. home.Name .. " " .. homeScore .. "-" .. awayScore .. " " .. away.Name) end
	else
		home.LeagueStats.Draws += 1 home.LeagueStats.Points += 1
		away.LeagueStats.Draws += 1 away.LeagueStats.Points += 1
		pushResult(home, "D") pushResult(away, "D")
	end

	-- Morale
	local function shiftMorale(club, delta)
		for _, p in pairs(club.Roster) do p.Morale = math.clamp(p.Morale + delta, 0, 100) end
	end
	if homeScore > awayScore then shiftMorale(home, 3) shiftMorale(away, -3)
	elseif awayScore > homeScore then shiftMorale(away, 3) shiftMorale(home, -3)
	else shiftMorale(home, 1) shiftMorale(away, 1) end
end

-- ================================================
-- 🌍 GLOBAL MATCH SIMULATION (Weekly)
-- ================================================
function WorldStateService:SimulateGlobalMatches(skipClubID)
	print("⚽ [SIM] Simulating global match week " .. self.Universe.Week .. "...")
	local regions = {"England","Spain","Germany","France","Italy"}
	for _, region in ipairs(regions) do
		for div = 1, 10 do
			local clubs = {}
			for _, c in pairs(self.Universe.Clubs) do
				if c.Region == region and c.Division == div then
					table.insert(clubs, c)
				end
			end
			-- Shuffle for random pairings
			for i = #clubs, 2, -1 do
				local j = math.random(i)
				clubs[i], clubs[j] = clubs[j], clubs[i]
			end
			for i = 1, #clubs - 1, 2 do
				local h, a = clubs[i], clubs[i+1]
				if not h or not a then break end
				if h.ID == skipClubID or a.ID == skipClubID then continue end

				-- 🎲 ENHANCED RANDOMIZATION ENGINE
				local hSkill = h.OVR + (h.Form * 3)
				local aSkill = a.OVR + (a.Form * 3)

				-- Upset chance: 15% chance of a major logic swing
				local noise = math.random(-12, 12)
				if math.random(1, 100) <= 15 then noise = math.random(-25, 25) end

				local hPow = hSkill + noise
				local aPow = aSkill

				local hGoals = math.random(0, 2)
				local aGoals = math.random(0, 2)

				if hPow > aPow + 15 then hGoals += math.random(1,3)
				elseif hPow > aPow + 5 then hGoals += math.random(0,2)
				elseif aPow > hPow + 15 then aGoals += math.random(1,3)
				elseif aPow > hPow + 5 then aGoals += math.random(0,2) end

				self:RecordMatchResult(h.ID, a.ID, hGoals, aGoals, true)
			end
		end
	end
end

-- ================================================
-- 📈 PLAYER DEVELOPMENT (Weekly)
-- ================================================
function WorldStateService:UpdatePlayerDevelopment()
	local eco = DIVISION_ECONOMY

	for clubID, club in pairs(self.Universe.Clubs) do

		-- 1. Recalculate wage bill using OVR-based formula
		local wageBill = 0
		for _, player in pairs(club.Roster) do
			player.Wage = self:CalcWage(player.OVR)
			wageBill += player.Wage
		end
		club.WageBill = wageBill

		-- 2. Apply weekly income (match day + sponsor)
		local divEco = eco[club.Division] or eco[10]
		club.Budget += divEco.matchIncome
		club.Budget += (club.SponsorIncome or SPONSOR_TIERS[club.Division] or 5000)

		-- 3. Deduct wages
		club.Budget -= wageBill

		-- 4. Financial pressure rule (Expenses > Income)
		local weeklyIncome   = divEco.matchIncome + (club.SponsorIncome or 0)
		local weeklyExpenses = wageBill
		if weeklyExpenses > weeklyIncome then
			-- Morale -2, BoardConfidence -3 per week
			club.Morale = math.max(0, (club.Morale or 75) - 2)
			club.BoardConfidence = math.max(0, (club.BoardConfidence or 50) - 3)
			self:PushNews("⚠️ " .. club.Name .. " are under financial pressure!")
		end

		-- 5. Budget floor warning
		if club.Budget < 0 then
			club.BoardConfidence = math.max(0, (club.BoardConfidence or 50) - 5)
			self:PushNews("🔴 " .. club.Name .. " are in serious financial difficulty!")
		end

		-- 6. Player development loop
		for pID, player in pairs(club.Roster) do
			-- Morale drift toward baseline
			player.Morale = math.clamp(player.Morale + (club.Form > 0 and 1 or -1), 0, 100)
			-- Youth growth (under 24)
			if player.Age < 24 and player.OVR < player.Potential then
				local growthChance = math.random(1, 10)
				local bonus = (club.Facilities and club.Facilities.Training or 1) > 3 and 2 or 1
				if growthChance <= bonus then
					player.OVR = math.min(player.OVR + 1, player.Potential)
					player.Value = math.max(50000, math.floor((player.OVR ^ 2) * 500))
				end
			end
			-- Decline (over 32)
			if player.Age > 32 and math.random(1, 15) == 1 then
				player.OVR = math.max(40, player.OVR - 1)
				player.Value = math.max(10000, math.floor(player.Value * 0.9))
			end
		end
		club.WageBill = wageBill

		-- Academy generation
		if #club.YouthAcademy < 5 then
			self:GenerateYouthProspects(clubID)
		end
	end
end

-- ================================================
-- 🎓 YOUTH ACADEMY
-- ================================================
function WorldStateService:GenerateYouthProspects(clubID)
	local club = self.Universe.Clubs[clubID]
	if not club then return end
	local level = club.Facilities.Youth or 1
	local count = math.random(1, 2 + math.floor(level / 3))
	local positions = {"ST","CM","CB","GK","LW","RW","CAM","CDM"}
	local firstNames = {"Alfie","Oliver","Ethan","Noah","Tyler","Marcus","Jamie","Liam","Jake","Aaron"}
	local lastNames  = {"Webb","Reece","Banks","Dixon","Foley","Marsh","Shaw","Hart","Burke","Owen"}

	for i = 1, count do
		local pOVR = math.clamp(35 + (level * 2) + math.random(-5, 5), 30, 60)
		local pID  = clubID .. "_Y" .. tostring(#club.YouthAcademy + 1) .. tostring(math.random(100,999))
		local pos  = positions[math.random(#positions)]

		table.insert(club.YouthAcademy, {
			ID          = pID,
			Name        = FIRST_NAMES.WESTERN[math.random(#FIRST_NAMES.WESTERN)] .. " " .. LAST_NAMES.ENGLISH[math.random(#LAST_NAMES.ENGLISH)],
			Age         = math.random(14, 18),
			Position    = pos,
			OVR         = pOVR,
			Stats       = randomizeStats(pOVR, pos),
			Potential   = math.random(pOVR + 10, 96),
			Morale      = 90,
			Condition   = 100,
			IsYouth     = true,
			NationalityFlag = "🏴󠁧󠁢󠁥󠁮󠁧󠁿",
		})
	end
end

-- ================================================
-- 💸 AI TRANSFERS (Weekly)
-- ================================================
function WorldStateService:ProcessAITransfers()
	for clubID, club in pairs(self.Universe.Clubs) do
		if club.ManagerName and club.ManagerName ~= "Manager" then continue end
		if club.Budget < 2000000 then continue end

		local rosterCount = 0
		for _ in pairs(club.Roster) do rosterCount += 1 end
		if rosterCount >= 25 then continue end

		-- Look for a free agent to sign
		for pID, player in pairs(self.Universe.Players) do
			if player.Club == "Free Agent" and player.Value <= club.Budget * 0.3 then
				if player.OVR > (club.OVR - 5) then
					player.Club = club.Name
					club.Budget -= player.Value
					club.Roster[pID] = player
					club.WageBill += player.Wage
					self:PushNews("📰 " .. club.Name .. " sign " .. player.Name)
					break
				end
			end
		end
	end
end

-- ================================================
-- 🌐 PROMOTION / RELEGATION
-- ================================================
function WorldStateService:ProcessPromotionRelegation()
	local regions = {"England","Spain","Germany","France","Italy"}
	for _, region in ipairs(regions) do
		for div = 1, 9 do -- No promotion from Div 1, no relegation from Div 10
			local standings = self:GetDivisionStandings(region, div)
			-- Bottom 3 of this division get relegated (go to div+1)
			for i = #standings - 2, #standings do
				if standings[i] and standings[i].Division == div then
					standings[i].Division += 1
					local eco = DIVISION_ECONOMY[standings[i].Division] or DIVISION_ECONOMY[10]
					standings[i].Budget = math.min(standings[i].Budget, eco.budget)
					standings[i].MatchIncome = eco.matchIncome
					standings[i].SponsorIncome = SPONSOR_TIERS[standings[i].Division] or 5000
					self:PushNews("⬇️ " .. standings[i].Name .. " relegated to Division " .. standings[i].Division)
				end
			end
		end
		for div = 2, 10 do -- Top 3 of this division get promoted
			local standings = self:GetDivisionStandings(region, div)
			for i = 1, math.min(3, #standings) do
				if standings[i] and standings[i].Division == div then
					standings[i].Division -= 1
					local eco = DIVISION_ECONOMY[standings[i].Division] or DIVISION_ECONOMY[1]
					standings[i].Budget = eco.budget
					standings[i].MatchIncome = eco.matchIncome
					standings[i].SponsorIncome = SPONSOR_TIERS[standings[i].Division] or 5000000
					-- Reset league stats for next season
					standings[i].LeagueStats = {Played=0,Wins=0,Draws=0,Losses=0,Points=0,GF=0,GA=0}
					self:PushNews("⬆️ " .. standings[i].Name .. " promoted to Division " .. standings[i].Division)
				end
			end
		end
	end
end

-- ================================================
-- 📰 NEWS FEED
-- ================================================
function WorldStateService:PushNews(headline)
	table.insert(self.Universe.NewsFeed, 1, { Headline = headline, Week = self.Universe.Week, Season = self.Universe.Season })
	if #self.Universe.NewsFeed > 100 then table.remove(self.Universe.NewsFeed, 101) end
end

-- ================================================
-- 💸 TRANSFER RECORDS
-- ================================================
function WorldStateService:AddTransferRecord(player, from, to, price)
	table.insert(self.Universe.TransferHistory, 1, {
		Player = player.Name,
		From = from,
		To = to,
		Price = price,
		Week = self.Universe.Week
	})
	if #self.Universe.TransferHistory > 50 then table.remove(self.Universe.TransferHistory, 51) end
end

return WorldStateService

________________________________________________

-- ================================================
-- SCOUT SERVICE (ServerScriptService) v3.0
-- ================================================
-- Logic for time-based mission scouting and 
-- hidden potential discovery.
-- ================================================

local WorldState = require(script.Parent.WorldStateService)

local ScoutService = {}

local MISSION_TYPES = {
	["Local"]    = { Duration = 1, Cost = 50000,   FindCount = 3, MaxOvr = 65 },
	["Regional"] = { Duration = 2, Cost = 250000,  FindCount = 5, MaxOvr = 75 },
	["Global"]   = { Duration = 4, Cost = 1000000, FindCount = 8, MaxOvr = 95 }
}

function ScoutService:StartMission(clubID, missionType)
	local club = WorldState.Universe.Clubs[clubID]
	if not club then return false, "Club not found" end

	local mData = MISSION_TYPES[missionType]
	if not mData then return false, "Invalid mission type" end

	if club.Budget < mData.Cost then return false, "Insufficient funds" end

	-- Deduction
	club.Budget -= mData.Cost

	local mission = {
		Type = missionType,
		StartTime = os.time(),
		Duration = mData.Duration * 60, -- 1 min per week for testing
		WeekAssigned = WorldState.Universe.Week,
		FindCount = mData.FindCount,
		MaxOvr = mData.MaxOvr,
		Status = "ACTIVE"
	}

	table.insert(club.ScoutMissions, mission)
	return true, mission
end

function ScoutService:Update(clubID)
	local club = WorldState.Universe.Clubs[clubID]
	if not club then return end

	for _, m in ipairs(club.ScoutMissions) do
		if m.Status == "ACTIVE" then
			local elapsed = os.time() - m.StartTime
			if elapsed >= m.Duration then
				m.Status = "COMPLETE"
				self:GenerateResults(club, m)
			end
		end
	end
end

function ScoutService:GenerateResults(club, mission)
	local results = {}
	for i = 1, mission.FindCount do
		-- Create a random player (Free Agent)
		local pID = "SCOUT_" .. club.ID .. "_" .. math.random(1000,9999)
		local ovr = math.random(40, mission.MaxOvr)
		local pot = math.clamp(ovr + math.random(0, 20), ovr, 99)

		local p = WorldState:CreatePlayer(pID, "Scouted Player " .. i, math.random(16,24), "CM", ovr, pot)
		table.insert(results, p)
	end
	mission.Results = results
	WorldState:PushNews("🔭 Scout report ready for " .. club.Name .. "!")
end

return ScoutService

___________________________________

AdmiNService

-- ================================================
-- ADMIN SERVICE (ServerScriptService - ModuleScript)
-- ================================================
-- Secure, server-authoritative admin system.
-- Role-based permissions: OWNER > ADMIN > VIEWER > PLAYER
-- OWNER: rxjuzes
-- ================================================
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players           = game:GetService("Players")

local DataManager       = require(script.Parent:WaitForChild("DataManager"))
local WorldStateService = require(script.Parent:WaitForChild("WorldStateService"))

local AdminService = {}

-- Roles enum
local ROLES = {
	OWNER  = 4,
	ADMIN  = 3,
	VIEWER = 2,
	PLAYER = 1,
}

-- Server-side only logs
local AdminLogs = {}

-- Remotes
local Remotes    = ReplicatedStorage:WaitForChild("Remotes")
local AdminFunc  = Remotes:WaitForChild("AdminFunc")
local AdminEvent = Remotes:WaitForChild("AdminEvent")

-- ================================================
-- 🔒 SECURITY HELPERS
-- ================================================
local function getRole(player)
	local p = DataManager:GetByPlayer(player)
	if not p then return ROLES.PLAYER end
	return ROLES[p.Role] or ROLES.PLAYER
end

local function isAllowed(player, minRole)
	local roleVal = getRole(player)
	return roleVal >= minRole
end

local function logAction(admin, action, target, details)
	local entry = {
		Admin     = admin.Name,
		Action    = action,
		Target    = target or "N/A",
		Details   = details or "",
		Timestamp = os.time(),
	}
	table.insert(AdminLogs, 1, entry)
	if #AdminLogs > 200 then table.remove(AdminLogs, 201) end
	-- Replication of logs to other Admins
	AdminEvent:FireAllClients("NewLog", entry)
end

-- ================================================
-- ⚙️ CORE ADMIN HANDLERS
-- ================================================
AdminFunc.OnServerInvoke = function(player, action, data)
	-- 💰 CHANGE MONEY (ADMIN+)
	if action == "SetMoney" then
		if not isAllowed(player, ROLES.ADMIN) then return false, "Unauthorized" end
		local targetName = data.Username
		local amount     = tonumber(data.Amount)
		if not targetName or not amount then return false, "Invalid data" end

		local targetPlayer = Players:FindFirstChild(targetName)
		if targetPlayer then
			local profile = DataManager:GetByPlayer(targetPlayer)
			if profile then
				profile.Budget = amount
				DataManager:Set(targetPlayer, profile)
				logAction(player, "SetMoney", targetName, "Set to £" .. amount)
				return true, "Success."
			end
		end
		return false, "Player Profile not found."

		-- 👀 CLUB VIEW (ADMIN+)
	elseif action == "GetClubView" then
		if not isAllowed(player, ROLES.ADMIN) then return false, "Unauthorized" end
		local targetName = data.Username
		local targetPlayer = Players:FindFirstChild(targetName)
		if targetPlayer then
			local profile = DataManager:GetByPlayer(targetPlayer)
			if profile and profile.HasClub then
				return true, profile
			end
			return false, "No Club Assigned."
		end
		return false, "Player not found."

		-- 🧠 GIVE ADMIN (OWNER ONLY)
	elseif action == "SetRole" then
		if not isAllowed(player, ROLES.OWNER) then return false, "Unauthorized" end
		local targetName = data.Username
		local newRole    = data.Role -- "ADMIN", "VIEWER", "PLAYER"
		if not targetName or not ROLES[newRole] then return false, "Invalid role" end

		local targetPlayer = Players:FindFirstChild(targetName)
		if targetPlayer then
			if targetPlayer.Name == "rxjuzes" then return false, "Cannot modify Owner." end
			local profile = DataManager:GetByPlayer(targetPlayer)
			if profile then
				profile.Role = newRole
				DataManager:Set(targetPlayer, profile)
				logAction(player, "SetRole", targetName, "Role set to " .. newRole)
				return true, "Success."
			end
		end
		return false, "Player not found."

		-- 🏟️ FORCE RESULT (ADMIN+)
	elseif action == "ForceResult" then
		-- ... (Logic to force a match score in WorldStateService)
		return false, "Not implemented yet."

		-- 🔁 WORLD RESET (OWNER ONLY)
	elseif action == "WorldReset" then
		if not isAllowed(player, ROLES.OWNER) then return false, "Unauthorized" end
		-- Implementation: iterate WorldState clubs and reset stats
		for _, club in pairs(WorldStateService.Universe.Clubs) do
			club.LeagueStats = {Played=0, Wins=0, Draws=0, Losses=0, Points=0, GF=0, GA=0}
			club.Budget = 1000000 -- Reset to standard base
		end
		WorldStateService.Universe.NewsFeed = {}
		logAction(player, "WorldReset", "Server", "Full World Stats Reset initiated.")
		return true, "World state reset complete."

		-- 📋 FETCH LOGS (VIEWER+)
	elseif action == "GetLogs" then
		if not isAllowed(player, ROLES.VIEWER) then return {} end
		return AdminLogs
	end
end

return AdminService

___________________________________________

-- ================================================
-- LEAGUE SERVICE (ServerScriptService) - v3.0 FINAL
-- ================================================
-- Handles: Club Creation, Career Mode Gate, Transfer Market,
--          Academy, Scouting, Facility Upgrades, Sponsors, Tactics
-- ================================================
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TextService       = game:GetService("TextService")
local WorldStateService = require(script.Parent:WaitForChild("WorldStateService"))

-- ================================================
-- 🛡️ SAFE REMOTE CREATION (uses Remotes folder from DataManager)
-- ================================================
local Remotes = ReplicatedStorage:WaitForChild("Remotes", 10)

local function getFunc(name)
	local f = Remotes:FindFirstChild(name)
	if f and f:IsA("RemoteFunction") then return f end
	-- Fallback: create if missing
	local n = Instance.new("RemoteFunction", Remotes); n.Name = name; return n
end

local function getEvent(name)
	local e = Remotes:FindFirstChild(name)
	if e and e:IsA("RemoteEvent") then return e end
	local n = Instance.new("RemoteEvent", Remotes); n.Name = name; return n
end

-- Events
local MatchUpdate     = getEvent("MatchUpdate")
local StartMatch      = getEvent("StartMatch")
local SimulateMatch   = getEvent("SimulateMatch")
local StatUpdate      = getEvent("StatUpdate")
local TrainingRequest = getEvent("TrainingRequest")

-- Functions
local SetCareerMode    = getFunc("SetCareerMode")
local GetLeagueData    = getFunc("GetLeagueData")
local GetTransferMarket= getFunc("GetTransferMarket")
local GetAcademyData   = getFunc("GetAcademyData")
local PromoteYouth     = getFunc("PromoteYouth")
local ReleaseYouth     = getFunc("ReleaseYouth")
local UpgradeFacility  = getFunc("UpgradeFacility")
local SendScout        = getFunc("SendScout")
local GetSponsorData   = getFunc("GetSponsorData")
local ClaimSponsor     = getFunc("ClaimSponsor")
local GetWorldNews     = getFunc("GetWorldNews")
local SetTactics       = getFunc("SetTactics")
local GetNextMatch     = getFunc("GetNextMatch")

-- ================================================
-- 📦 STATE
-- ================================================
local LeagueService = {}
local PlayerClubs   = {}   -- [userId] = clubID
local ScoutCooldowns= {}   -- [userId] = tick()
local SCOUT_COOLDOWN = 60  -- seconds before scouting returns results

-- Sponsor pool
local SPONSORS = {
	{ Name = "NeoCola®",    Value = function(div) return 500000  * (11 - div) end, Goal = "Win 3 matches in a row",     GoalType = "WinStreak", GoalAmount = 3  },
	{ Name = "Apex Gear",   Value = function(div) return 1000000 * (11 - div) end, Goal = "Reach the Top 4",            GoalType = "Position",  GoalAmount = 4  },
	{ Name = "CyberBank",   Value = function(div) return 1500000 * (11 - div) end, Goal = "Promote a youth player",     GoalType = "PromoteYouth", GoalAmount = 1 },
	{ Name = "FutureFit",   Value = function(div) return 700000  * (11 - div) end, Goal = "Keep a clean sheet 5 times", GoalType = "CleanSheet",GoalAmount = 5  },
	{ Name = "StarStrike",  Value = function(div) return 1200000 * (11 - div) end, Goal = "Score 20 goals in a season", GoalType = "Goals",     GoalAmount = 20 },
	{ Name = "OmegaKit",    Value = function(div) return 800000  * (11 - div) end, Goal = "Win 5 home matches",         GoalType = "HomeWins",  GoalAmount = 5  },
}

-- Formation table
local FORMATIONS = {
	"4-3-3","4-4-2","4-2-3-1","3-5-2","5-3-2","3-4-3","4-5-1","4-1-4-1"
}
local PLAYSTYLES = {
	"Possession","Counter-Attack","High Press","Low Block","Direct Play","Tiki-Taka","Park the Bus"
}

-- ================================================
-- 🌐 NAME FILTER
-- ================================================
local function filterName(player, text)
	local clean = text
	pcall(function()
		local res = TextService:FilterStringAsync(text, player.UserId)
		clean = res:GetNonChatStringForBroadcastAsync()
	end)
	if clean ~= text then return false, clean end
	-- Pattern-block bad words
	local banned = {"ass","fuck","shit","bitch","dick","cunt","nigger","retard","faggot","porn","cock","slut","whore","rape","nazi","hitler","kys"}
	local lower = " " .. string.lower(text) .. " "
	for _, word in ipairs(banned) do
		if string.find(lower, "[^%a]" .. word .. "[^%a]") then return false, clean end
	end
	return true, clean
end

-- ================================================
-- 🏟️ CAREER SETUP
-- ================================================
SetCareerMode.OnServerInvoke = function(player, clubName, shortName, managerName, region, division, kitHome, kitAway, logoShape, logoIcon)
	-- Validate
	if type(clubName) ~= "string" or #clubName < 2 or #clubName > 22 then return false, "Invalid club name length" end
	if type(shortName) ~= "string" or #shortName < 2 or #shortName > 5 then return false, "Invalid short name length" end
	if string.match(clubName, "[^%a%d%s]") then return false, "Letters and numbers only" end

	local nameOk, safeName   = filterName(player, clubName)
	local shortOk, safeShort = filterName(player, shortName)
	if not nameOk or not shortOk then return false, "Inappropriate name" end

	division = math.clamp(tonumber(division) or 10, 1, 10)
	region = (region == "England" or region == "Spain" or region == "Germany"
		or region == "France" or region == "Italy" or region == "Rest of World")
		and region or "England"

	-- Remove one AI club from that division to make room
	local replaced = false
	for id, c in pairs(WorldStateService.Universe.Clubs) do
		if c.Region == region and c.Division == division and c.ManagerName == "Manager" then
			WorldStateService.Universe.Clubs[id] = nil
			replaced = true
			break
		end
	end

	local clubID  = tostring(player.UserId) .. "_Club"

	-- ✅ Seed correct budget by division (spec-accurate economy)
	local STARTING_BUDGETS = {
		[1]=250000000, [2]=120000000, [3]=60000000,  [4]=30000000, [5]=15000000,
		[6]=8000000,   [7]=4000000,   [8]=2000000,   [9]=1000000,  [10]=500000
	}
	local STARTING_INCOME = {
		[1]=2000000, [2]=900000, [3]=650000, [4]=400000, [5]=250000,
		[6]=150000,  [7]=100000, [8]=60000,  [9]=35000,  [10]=20000
	}

	local startBudget = STARTING_BUDGETS[division] or 500000
	local newClub  = WorldStateService:CreateClub(clubID, safeName, region, division, startBudget, "Balanced")
	newClub.ShortName    = string.upper(safeShort):sub(1,5)
	newClub.ManagerName  = (type(managerName) == "string" and #managerName > 0) and managerName or player.Name
	newClub.Kits         = { Home = kitHome or {R=0,G=100,B=220}, Away = kitAway or {R=255,G=255,B=255}, Third = {R=0,G=0,B=0} }
	newClub.Logo         = { Shape = logoShape or "Shield", Icon = logoIcon or "⚽" }
	newClub.MatchIncome  = STARTING_INCOME[division] or 20000
	newClub.SponsorIncome= SPONSOR_TIERS and SPONSOR_TIERS[division] or 5000

	-- Generate starting roster based on division
	local baseOVR = math.clamp(90 - (division * 7), 40, 85)
	WorldStateService:GenerateRoster(clubID, region, baseOVR)

	-- Assign 3 random sponsors
	local usedSponsors = {}
	while #newClub.Sponsors < 3 do
		local idx = math.random(#SPONSORS)
		if not usedSponsors[idx] then
			local s = SPONSORS[idx]
			table.insert(newClub.Sponsors, {
				Name       = s.Name,
				Value      = s.Value(division),
				Goal       = s.Goal,
				GoalType   = s.GoalType,
				GoalAmount = s.GoalAmount,
				Progress   = 0,
				Claimed    = false,
			})
			usedSponsors[idx] = true
		end
	end

	PlayerClubs[player.UserId] = clubID

	-- 💾 Persist to DataManager so it survives server restarts
	local DataManager = require(script.Parent:WaitForChild("DataManager"))
	local profile = DataManager:GetByPlayer(player)
	if profile then
		profile.ClubName      = newClub.Name
		profile.ShortName     = newClub.ShortName
		profile.Region        = newClub.Region
		profile.Division      = newClub.Division
		profile.Budget        = newClub.Budget
		profile.Sponsors      = newClub.Sponsors
		profile.Formation     = newClub.Formation
		profile.Playstyle     = newClub.Playstyle
		profile.HasClub       = true
		profile.Finances      = {
			Income   = newClub.MatchIncome,
			Expenses = 0,
			WageBill = newClub.WageBill or 0,
		}
		DataManager:Set(player, profile)
	end

	WorldStateService:PushNews("🏆 " .. safeName .. " has entered the " .. region .. " Division " .. division .. "!")
	print("✅ [LEAGUE] Career started: " .. safeName .. " | " .. region .. " Div " .. division .. " | Budget: £" .. startBudget)
	return true, "OK"
end

-- ================================================
-- 📋 DATA BRIDGE
-- ================================================
local function getPlayerClub(player)
	local clubID = PlayerClubs[player.UserId]
	return clubID, clubID and WorldStateService.Universe.Clubs[clubID]
end

GetLeagueData.OnServerInvoke = function(player)
	local clubID, myClub = getPlayerClub(player)
	if not myClub then return nil end
	return {
		ClubID       = clubID,
		ClubName     = myClub.Name,
		ShortName    = myClub.ShortName,
		ManagerName  = myClub.ManagerName,
		Region       = myClub.Region,
		Division     = myClub.Division,
		DivisionName = "Division " .. myClub.Division,
		Budget       = myClub.Budget,
		WageBill     = myClub.WageBill,
		BoardConfidence = myClub.BoardConfidence or 50,
		Facilities   = myClub.Facilities,
		Standings    = WorldStateService:GetDivisionStandings(myClub.Region, myClub.Division),
		Roster       = myClub.Roster,
		RecentResults= myClub.RecentResults,
		Formation    = myClub.Formation,
		Playstyle    = myClub.Playstyle,
		StartingXI   = myClub.StartingXI,
		OVR          = myClub.OVR,
		Season       = WorldStateService.Universe.Season,
		Week         = WorldStateService.Universe.Week,
	}
end

GetNextMatch.OnServerInvoke = function(player)
	local clubID, myClub = getPlayerClub(player)
	if not myClub then return nil end

	local week = WorldStateService.Universe.Week
	local fixtures = WorldStateService.Universe.Fixtures[myClub.Region] 
		and WorldStateService.Universe.Fixtures[myClub.Region][myClub.Division]

	if not fixtures or not fixtures[week] then return nil end

	-- Find which pairing in this week's fixture list includes our club
	for _, match in ipairs(fixtures[week]) do
		local oppID
		if match.Home == clubID then oppID = match.Away
		elseif match.Away == clubID then oppID = match.Home end

		if oppID then
			local opp = WorldStateService.Universe.Clubs[oppID]
			if opp then
				return { 
					OpponentName = opp.Name, 
					OpponentOVR  = opp.OVR, 
					Division     = myClub.Division,
					IsHome       = (match.Home == clubID)
				}
			end
		end
	end
	return nil
end

-- ================================================
-- 🔁 TRANSFER MARKET
-- ================================================
GetTransferMarket.OnServerInvoke = function(player)
	local list = {}
	-- Pull real free agents from world state
	for pID, p in pairs(WorldStateService.Universe.Players) do
		if p.Club == "Free Agent" then
			table.insert(list, p)
		end
	end
	-- Fill with generated ones if low
	local WorldGenerator = require(game:GetService("ReplicatedStorage"):WaitForChild("WorldGenerator", 3))
	if WorldGenerator then
		while #list < 20 do
			local fa = WorldGenerator:GenerateFreeAgent()
			local id = "FA_" .. tostring(math.random(100000,999999))
			fa.ID = id
			WorldStateService.Universe.Players[id] = fa
			table.insert(list, fa)
		end
	end
	-- Sort by OVR desc
	table.sort(list, function(a,b) return a.OVR > b.OVR end)
	return list
end

-- ================================================
-- 🎓 ACADEMY
-- ================================================
GetAcademyData.OnServerInvoke = function(player)
	local _, myClub = getPlayerClub(player)
	if not myClub then return {} end
	return myClub.YouthAcademy
end

PromoteYouth.OnServerInvoke = function(player, youthID)
	local _, myClub = getPlayerClub(player)
	if not myClub then return false, "No club" end
	for i, p in ipairs(myClub.YouthAcademy) do
		if p.ID == youthID then
			p.Club  = myClub.Name
			p.Value = math.max(50000, p.OVR * 300000)
			p.Wage  = math.floor(p.OVR * 30)
			myClub.Roster[p.ID] = p
			myClub.WageBill += p.Wage
			table.remove(myClub.YouthAcademy, i)
			-- Update sponsor progress
			for _, s in ipairs(myClub.Sponsors) do
				if s.GoalType == "PromoteYouth" and not s.Claimed then
					s.Progress += 1
				end
			end
			WorldStateService:PushNews("🎓 " .. myClub.Name .. " promote youth star " .. p.Name .. " to the first team!")
			print("🎓 Promoted: " .. p.Name)
			return true, "Promoted"
		end
	end
	return false, "Player not found"
end

ReleaseYouth.OnServerInvoke = function(player, youthID)
	local _, myClub = getPlayerClub(player)
	if not myClub then return false end
	for i, p in ipairs(myClub.YouthAcademy) do
		if p.ID == youthID then
			table.remove(myClub.YouthAcademy, i)
			return true
		end
	end
	return false
end

-- ================================================
-- 🔭 SCOUTING
-- ================================================
local ScoutResults = {}  -- [userId] = { players, returnAt }

SendScout.OnServerInvoke = function(player, region, position, minAge, maxAge)
	local _, myClub = getPlayerClub(player)
	if not myClub then return false, "No club" end

	local now = tick()
	local pending = ScoutResults[player.UserId]
	-- If results exist and cooldown passed, return them
	if pending and now >= pending.returnAt then
		local res = pending.players
		ScoutResults[player.UserId] = nil  -- clear after retrieval
		return true, res
	end
	-- If currently scouting, return "pending"
	if pending and now < pending.returnAt then
		return false, math.ceil(pending.returnAt - now)
	end

	-- Send a new scout
	local positions  = {"ST","LW","RW","CAM","CM","CDM","CB","LB","RB","GK"}
	local targetPos  = (position and position ~= "ANY") and position or positions[math.random(#positions)]
	local discovered = {}

	for i = 1, 4 + (myClub.Facilities.Training or 1) do
		local age = math.random(minAge or 16, maxAge or 28)
		local ovr = math.random(55, 80)
		local pot = math.min(99, ovr + math.random(5, 25))
		local pID = "SCOUT_" .. tostring(math.random(100000,999999))
		local p   = WorldStateService:CreatePlayer(pID, "Scout #" .. i, age, targetPos, ovr, pot, "Balanced")
		p.Club    = "Free Agent"
		table.insert(discovered, p)
	end

	-- Cooldown = 45 seconds base, reduced by Youth facility level
	local returnTime = now + math.max(15, SCOUT_COOLDOWN - (myClub.Facilities.Youth or 1) * 5)
	ScoutResults[player.UserId] = { players = discovered, returnAt = returnTime }

	return false, math.ceil(returnTime - now)   -- returns false + seconds to wait
end

-- ================================================
-- 🏋️ FACILITY UPGRADES
-- ================================================
UpgradeFacility.OnServerInvoke = function(player, facilityType)
	local _, myClub = getPlayerClub(player)
	if not myClub then return false, "No club" end
	if not ({ Training=true, Youth=true, Recovery=true })[facilityType] then return false, "Invalid facility" end

	local level    = myClub.Facilities[facilityType] or 1
	if level >= 10 then return false, "Max level" end

	local cost = level * 2500000
	if myClub.Budget < cost then return false, "Insufficient budget" end

	myClub.Budget -= cost
	myClub.Facilities[facilityType] = level + 1
	WorldStateService:PushNews("🏋️ " .. myClub.Name .. " upgrade " .. facilityType .. " facility to level " .. level+1)
	return true, "Upgraded to level " .. (level+1)
end

-- ================================================
-- 📣 SPONSORS
-- ================================================
GetSponsorData.OnServerInvoke = function(player)
	local _, myClub = getPlayerClub(player)
	if not myClub then return {} end
	return myClub.Sponsors
end

ClaimSponsor.OnServerInvoke = function(player, sponsorName)
	local _, myClub = getPlayerClub(player)
	if not myClub then return false, "No club" end
	for _, s in ipairs(myClub.Sponsors) do
		if s.Name == sponsorName and not s.Claimed and s.Progress >= s.GoalAmount then
			s.Claimed = true
			myClub.Budget += s.Value
			WorldStateService:PushNews("💰 " .. myClub.Name .. " claim " .. s.Name .. " sponsorship reward!")
			return true, s.Value
		end
	end
	return false, "Conditions not met"
end

-- ================================================
-- 🗞️ WORLD NEWS
-- ================================================
GetWorldNews.OnServerInvoke = function(player)
	return WorldStateService.Universe.NewsFeed
end

-- ================================================
-- ⚙️ TACTICS
-- ================================================
SetTactics.OnServerInvoke = function(player, formation, playstyle, startingXI)
	local _, myClub = getPlayerClub(player)
	if not myClub then return false end

	-- Validate formation
	local validFormation = false
	for _, f in ipairs(FORMATIONS) do if f == formation then validFormation = true break end end
	if not validFormation then formation = myClub.Formation end

	-- Validate playstyle
	local validPlaystyle = false
	for _, p in ipairs(PLAYSTYLES) do if p == playstyle then validPlaystyle = true break end end
	if not validPlaystyle then playstyle = myClub.Playstyle end

	myClub.Formation  = formation
	myClub.Playstyle  = playstyle
	if type(startingXI) == "table" then
		myClub.StartingXI = startingXI
	end
	return true
end

-- ================================================
-- 🏟️ NEXT MATCH LOOKUP
-- ================================================
GetNextMatch.OnServerInvoke = function(player)
	local _, myClub = getPlayerClub(player)
	if not myClub then return nil end

	local week = WorldStateService.Universe.Week
	local reg  = myClub.Region
	local div  = myClub.Division

	local fixtures = WorldStateService.Universe.Fixtures[reg] and WorldStateService.Universe.Fixtures[reg][div]
	if not fixtures then return nil end

	local weekFixtures = fixtures[week]
	if not weekFixtures then return nil end

	for _, fix in ipairs(weekFixtures) do
		local oppID = ""
		if fix.Home == myClub.ID then oppID = fix.Away
		elseif fix.Away == myClub.ID then oppID = fix.Home end

		if oppID ~= "" then
			local opp = WorldStateService.Universe.Clubs[oppID]
			if opp then
				return {
					OpponentName = opp.Name,
					OpponentID   = oppID,
					OpponentOVR  = opp.OVR,
					IsHome       = (fix.Home == myClub.ID),
					Week         = week
				}
			end
		end
	end
	return nil
end

-- ================================================
-- 📊 LEAGUE & WORLD DATA
-- ================================================
GetLeagueData.OnServerInvoke = function(player, scope, region, division)
	-- scope: "Domestic", "Continental", "Global"
	if scope == "Global" then
		-- Return top 50 clubs globally by OVR/Points
		local all = {}
		for _, c in pairs(WorldStateService.Universe.Clubs) do table.insert(all, c) end
		table.sort(all, function(a, b) 
			local aP = a.LeagueStats.Points or 0
			local bP = b.LeagueStats.Points or 0
			if aP ~= bP then return aP > bP end
			return (a.OVR or 0) > (b.OVR or 0)
		end)
		local res = {}
		for i=1, 50 do if all[i] then table.insert(res, all[i]) end end
		return res
	elseif scope == "Continental" then
		-- Return top clubs in specified region
		local all = {}
		for _, c in pairs(WorldStateService.Universe.Clubs) do 
			if c.Region == region then table.insert(all, c) end 
		end
		table.sort(all, function(a, b) 
			local aP = a.LeagueStats.Points or 0
			local bP = b.LeagueStats.Points or 0
			if aP ~= bP then return aP > bP end
			local aGD = a.LeagueStats.GF - a.LeagueStats.GA
			local bGD = b.LeagueStats.GF - b.LeagueStats.GA
			if aGD ~= bGD then return aGD > bGD end
			return (a.OVR or 0) > (b.OVR or 0)
		end)
		local res = {}
		for i=1, 20 do if all[i] then table.insert(res, all[i]) end end
		return res
	else
		-- Default: Domestic Division
		return WorldStateService:GetDivisionStandings(region or "England", division or 1)
	end
end

-- ================================================
-- 🌍 WORLD GENERATOR
-- ================================================
function LeagueService:GenerateWorld()
	local regions   = {"England","Spain","Germany","France","Italy"}
	local cityDB = {
		England  = {"London","Manchester","Liverpool","Birmingham","Leeds","Newcastle","Sheffield","Leicester"},
		Spain    = {"Madrid","Barcelona","Valencia","Seville","Bilbao","Malaga","Granada","Zaragoza"},
		Germany  = {"Berlin","Munich","Hamburg","Dortmund","Frankfurt","Cologne","Leipzig","Stuttgart"},
		France   = {"Paris","Lyon","Marseille","Lille","Monaco","Nantes","Bordeaux","Nice"},
		Italy    = {"Rome","Milan","Naples","Turin","Florence","Genoa","Palermo","Bologna"},
	}
	local suffixes = {"FC","United","City","Athletic","Wanderers","Rovers","Town","Stars","Lions","Warriors"}

	for _, region in ipairs(regions) do
		local cities = cityDB[region] or cityDB["England"]
		local usedNames = {}

		for div = 1, 10 do
			local baseOVR = math.clamp(90 - (div * 5), 40, 88)

			for i = 1, 20 do
				local clubID = region .. "_D" .. div .. "_C" .. i

				-- Unique club name
				local name, attempts = "", 0
				repeat
					local city = cities[math.random(#cities)]
					local suf  = suffixes[math.random(#suffixes)]
					name = city .. " " .. suf
					if i > 15 then name = name .. " " .. i end
					attempts += 1
				until not usedNames[name] or attempts > 30
				usedNames[name] = true

				local club = WorldStateService:CreateClub(clubID, name, region, div, nil, "Balanced")
				WorldStateService:GenerateRoster(clubID, region, baseOVR)
			end
		end
	end

	-- Generate 50 global free agents
	local WorldGenerator = require(game:GetService("ReplicatedStorage"):WaitForChild("WorldGenerator", 3))
	if WorldGenerator then
		for i = 1, 50 do
			local fa   = WorldGenerator:GenerateFreeAgent()
			local faID = "FA_" .. tostring(i)
			fa.ID = faID
			WorldStateService.Universe.Players[faID] = fa
		end
	end

	-- 📅 Generate Season Fixtures
	for _, reg in ipairs(regions) do
		for div = 1, 10 do
			WorldStateService:GenerateFixturesForDivision(reg, div)
		end
	end

	print("✅ [LEAGUE] World generated: " .. #regions .. " regions × 10 divisions × 20 clubs + fixtures generated.")
end

LeagueService:GenerateWorld()
return LeagueService


_____________________________________________

Datamanager

-- ================================================
-- DATA MANAGER (ServerScriptService — ModuleScript) v3.0
-- ================================================
-- Server-authoritative. Stores full ClubProfile per player.
-- Uses UpdateAsync + 3-attempt retry. 60s autosave loop.
-- Client pulls via RequestData RemoteFunction.
-- ================================================

local DataStoreService  = game:GetService("DataStoreService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players           = game:GetService("Players")

local STORE_KEY   = "FootballManager_v3"
local CareerStore = DataStoreService:GetDataStore(STORE_KEY)

-- ================================================
-- REMOTES SETUP (server always creates these)
-- ================================================
local Remotes = ReplicatedStorage:FindFirstChild("Remotes")
if not Remotes then
	Remotes        = Instance.new("Folder")
	Remotes.Name   = "Remotes"
	Remotes.Parent = ReplicatedStorage
end

local function ensureRemote(class, name)
	local existing = Remotes:FindFirstChild(name)
	if existing and existing:IsA(class) then return existing end
	if existing then existing:Destroy() end
	local r = Instance.new(class)
	r.Name   = name
	r.Parent = Remotes
	return r
end

local RequestData = ensureRemote("RemoteFunction", "RequestData")
local UpdateData  = ensureRemote("RemoteEvent",    "UpdateData")
local UIEvent     = ensureRemote("RemoteEvent",    "UIEvent")
local AdminEvent  = ensureRemote("RemoteEvent",    "AdminEvent") -- [FIX]
local AdminFunc   = ensureRemote("RemoteFunction", "AdminFunc")  -- [FIX]

-- Keep legacy remotes alive so existing LeagueService/UIClient don't error
ensureRemote("RemoteEvent",    "StatUpdate")
ensureRemote("RemoteEvent",    "MatchUpdate")
ensureRemote("RemoteEvent",    "StartMatch")
ensureRemote("RemoteEvent",    "SimulateMatch")
ensureRemote("RemoteEvent",    "TrainingRequest")
ensureRemote("RemoteFunction", "SetCareerMode")
ensureRemote("RemoteFunction", "GetLeagueData")
ensureRemote("RemoteFunction", "GetTransferMarket")
ensureRemote("RemoteFunction", "GetAcademyData")
ensureRemote("RemoteFunction", "PromoteYouth")
ensureRemote("RemoteFunction", "ReleaseYouth")
ensureRemote("RemoteFunction", "UpgradeFacility")
ensureRemote("RemoteFunction", "SendScout")
ensureRemote("RemoteFunction", "GetSponsorData")
ensureRemote("RemoteFunction", "ClaimSponsor")
ensureRemote("RemoteFunction", "GetWorldNews")
ensureRemote("RemoteFunction", "SetTactics")
ensureRemote("RemoteFunction", "GetNextMatch")

-- ================================================
-- DEFAULT CLUB PROFILE
-- ================================================
local function DefaultProfile()
	return {
		ClubName   = "",
		ShortName  = "",
		Region     = "",
		Division   = 10,
		Role       = "PLAYER",
		Budget     = 500000,

		Stats = {
			Wins   = 0,
			Losses = 0,
			Draws  = 0,
			Points = 0,
			GF     = 0,
			GA     = 0,
		},

		Squad        = {},
		YouthPlayers = {},
		Scouts       = {},

		Finances = {
			Income   = 20000,  -- Division 10 base weekly income
			Expenses = 0,
			WageBill = 0,
		},

		Sponsors        = {},
		Morale          = 75,
		BoardConfidence = 50,
		Formation       = "4-3-3",
		Playstyle       = "Possession",
		RecentResults   = {},

		HasClub      = false,
		TutorialDone = false,
		Season       = 2025,
		Week         = 1,
	}
end

-- ================================================
-- IN-MEMORY CACHE
-- ================================================
local Profiles = {}   -- [userId] = ClubProfile table

-- ================================================
-- SAVE with UpdateAsync + retry
-- ================================================
local function SaveProfile(player)
	local profile = Profiles[player.UserId]
	if not profile then return end

	local key      = "Player_" .. player.UserId
	local attempts = 0

	repeat
		attempts += 1
		local ok, err = pcall(function()
			CareerStore:UpdateAsync(key, function()
				return profile
			end)
		end)

		if ok then
			print("✅ [DATA] Saved " .. player.Name .. " (attempt " .. attempts .. ")")
			return
		else
			warn("⚠️ [DATA] Save attempt " .. attempts .. " failed for "
				.. player.Name .. ": " .. tostring(err))
			if attempts < 3 then task.wait(3) end
		end
	until attempts >= 3

	warn("❌ [DATA] All save attempts exhausted for " .. player.Name)
end

-- ================================================
-- LOAD with forward-compatibility patching
-- ================================================
local function LoadProfile(player)
	local key    = "Player_" .. player.UserId
	local ok, data = pcall(function()
		return CareerStore:GetAsync(key)
	end)

	local profile
	if ok and type(data) == "table" then
		-- Patch any missing fields
		local defaults = DefaultProfile()
		for k, v in pairs(defaults) do
			if data[k] == nil then data[k] = v end
		end
		profile = data
	else
		profile = DefaultProfile()
	end

	-- 👑 OWNER OVERRIDE (per spec)
	if player.Name == "rxjuzes" then
		profile.Role = "OWNER"
	end

	Profiles[player.UserId] = profile
	print("✅ [DATA] Loaded profile for " .. player.Name .. " (Role: " .. profile.Role .. ")")

	-- Push full profile to client immediately
	task.spawn(function()
		task.wait(1)  -- brief wait for client script to connect
		UpdateData:FireClient(player, "FullSync", profile)
	end)
end

-- ================================================
-- REMOTE: Client requests its profile
-- ================================================
RequestData.OnServerInvoke = function(player)
	local waited = 0
	while not Profiles[player.UserId] and waited < 5 do
		task.wait(0.1)
		waited += 0.1
	end
	return Profiles[player.UserId]
end

-- ================================================
-- PUBLIC API (used by LeagueService / Server.lua)
-- ================================================
local DataManager = {}

function DataManager:Get(userId)
	return Profiles[userId]
end

function DataManager:GetByPlayer(player)
	return Profiles[player.UserId]
end

-- Full profile replacement + client sync
function DataManager:Set(player, profile)
	Profiles[player.UserId] = profile
	UpdateData:FireClient(player, "FullSync", profile)
end

-- Update a single top-level field + notify client
function DataManager:PatchField(player, field, value)
	local p = Profiles[player.UserId]
	if not p then return end
	p[field] = value
	UpdateData:FireClient(player, "FieldUpdate", { Field = field, Value = value })
end

function DataManager:Save(player)
	SaveProfile(player)
end

function DataManager:LoadData(player)  LoadProfile(player) end  -- legacy alias
function DataManager:SaveData(player)  SaveProfile(player) end  -- legacy alias

-- ================================================
-- PLAYER LIFECYCLE
-- ================================================
Players.PlayerAdded:Connect(function(player)
	LoadProfile(player)
end)

Players.PlayerRemoving:Connect(function(player)
	SaveProfile(player)
	Profiles[player.UserId] = nil
end)

-- Handle players already in game (Studio play mode)
for _, player in ipairs(Players:GetPlayers()) do
	task.spawn(function() LoadProfile(player) end)
end

-- ================================================
-- AUTO-SAVE LOOP (every 60 seconds)
-- ================================================
task.spawn(function()
	while true do
		task.wait(60)
		for _, player in ipairs(Players:GetPlayers()) do
			task.spawn(function() SaveProfile(player) end)
		end
	end
end)

return DataManager


_____________________________________________

MatchService

-- ================================================
-- MATCH SERVICE (ServerScriptService - Script)
-- ================================================
-- PILLARS COVERED: 5 (Event-Based Sim), 6 (Decision Ticks)
-- Replaces old chaotic physical simulation with structured logic phases.

local ReplicatedStorage = game:GetService("ReplicatedStorage")
local WorldStateService = require(script.Parent:WaitForChild("WorldStateService"))

-- Remotes
local StartMatch = ReplicatedStorage:FindFirstChild("StartMatch") or Instance.new("RemoteEvent", ReplicatedStorage)
StartMatch.Name = "StartMatch"

local MatchUpdate = ReplicatedStorage:FindFirstChild("MatchUpdate") or Instance.new("RemoteEvent", ReplicatedStorage)
MatchUpdate.Name = "MatchUpdate"

-- ================================================
-- 📊 MATCH STATE
-- ================================================
local ActiveMatch = {
	IsRunning = false,
	Minute = 0,
	HomeClub = nil,
	AwayClub = nil,
	Score = {Home = 0, Away = 0},

	-- Engine State
	Possession = "Home", -- "Home" or "Away"
	Phase = "BuildUp",   -- "BuildUp", "Attack", "Chance", "Defense"
}

-- ================================================
-- 🧠 EVENT ENGINE (Decision Tick)
-- ================================================
local function ProcessTick()
	local attackingTeam = ActiveMatch.Possession == "Home" and ActiveMatch.HomeClub or ActiveMatch.AwayClub
	local defendingTeam = ActiveMatch.Possession == "Home" and ActiveMatch.AwayClub or ActiveMatch.HomeClub

	-- Base OVRs (Pillar 5 Formula: OVR + Form)
	local attackOVR  = attackingTeam.OVR + (attackingTeam.Form or 0)
	local defenseOVR = defendingTeam.OVR + (defendingTeam.Form or 0)

	-- 🧮 DECISION TREE (Phase Based)
	if ActiveMatch.Phase == "Midfield" then
		-- 🏁 THE MIDFIELD BATTLE (New)
		local hMid = attackingTeam.OVR + math.random(-5, 5)
		local aMid = defendingTeam.OVR + math.random(-5, 5)

		if hMid > aMid - 5 then
			ActiveMatch.Phase = "BuildUp"
			MatchUpdate:FireAllClients(ActiveMatch, "Event", attackingTeam.Name .. " controls the midfield.")
		else
			ActiveMatch.Possession = (ActiveMatch.Possession == "Home") and "Away" or "Home"
			MatchUpdate:FireAllClients(ActiveMatch, "Event", "Midfield turnover! " .. defendingTeam.Name .. " wins the ball.")
		end

	elseif ActiveMatch.Phase == "BuildUp" then
		-- Trying to move into Attack
		local passSuccess = math.random(1, 100) + (attackOVR * 0.1)
		local interceptionChance = math.random(1, 100) + (defenseOVR * 0.1)

		if passSuccess > interceptionChance then
			ActiveMatch.Phase = "Attack"
			MatchUpdate:FireAllClients(ActiveMatch, "Event", attackingTeam.Name .. " advances into the attacking third.")
		else
			ActiveMatch.Phase = "Midfield"
			ActiveMatch.Possession = (ActiveMatch.Possession == "Home") and "Away" or "Home"
			MatchUpdate:FireAllClients(ActiveMatch, "Event", "Pass intercepted by " .. defendingTeam.Name .. ".")
		end

	elseif ActiveMatch.Phase == "Attack" then
		-- Trying to create a Chance
		local createChance = math.random(1, 100) + (attackOVR * 0.15)
		local tackleChance = math.random(1, 100) + (defenseOVR * 0.15)

		if createChance > tackleChance then
			ActiveMatch.Phase = "Chance"
			MatchUpdate:FireAllClients(ActiveMatch, "Event", "Dangerous attack building for " .. attackingTeam.Name .. "!")
		else
			ActiveMatch.Phase = "Midfield"
			ActiveMatch.Possession = (ActiveMatch.Possession == "Home") and "Away" or "Home"
			MatchUpdate:FireAllClients(ActiveMatch, "Event", "Great tackle by " .. defendingTeam.Name .. ".")
		end

	elseif ActiveMatch.Phase == "Chance" then
		-- 🎯 SHOT CALCULATION
		MatchUpdate:FireAllClients(ActiveMatch, "Event", attackingTeam.Name .. " takes a SHOT!")
		task.wait(0.5)

		local shotPower = math.random(50, 100) + (attackOVR * 0.2)
		local savePower = math.random(50, 100) + (defenseOVR * 0.2)

		if shotPower > savePower + 5 then
			-- ⚽ GOAL!
			ActiveMatch.Score[ActiveMatch.Possession] += 1
			MatchUpdate:FireAllClients(ActiveMatch, "Event", "⚽ GOAL for " .. attackingTeam.Name .. "!")
		else
			-- 🧤 SAVE / MISS
			MatchUpdate:FireAllClients(ActiveMatch, "Event", "🧤 Save! The " .. defendingTeam.Name .. " keeper holds on.")
		end

		-- Reset to midfield
		ActiveMatch.Phase = "Midfield"
		ActiveMatch.Possession = (ActiveMatch.Possession == "Home") and "Away" or "Home"
	end
end

-- ================================================
-- ⏱️ MAIN MATCH LOOP (Pillar 6)
-- ================================================
StartMatch.OnServerEvent:Connect(function(player)
	if ActiveMatch.IsRunning then return end

	-- 1. Identify official fixture for this week
	local DataManager = require(script.Parent:WaitForChild("DataManager"))
	local coreData    = DataManager:GetByPlayer(player)
	if not coreData or not coreData.HasClub then return end

	local week      = WorldStateService.Universe.Week
	local myClubID  = tostring(player.UserId) .. "_Club"
	local fixtures  = WorldStateService.Universe.Fixtures[coreData.Region] 
		and WorldStateService.Universe.Fixtures[coreData.Region][coreData.Division]

	if not fixtures or not fixtures[week] then 
		warn("❌ No fixture found for Week " .. week)
		return 
	end

	for _, m in ipairs(fixtures[week]) do
		if m.Home == myClubID or m.Away == myClubID then
			ActiveMatch.HomeClub = WorldStateService.Universe.Clubs[m.Home]
			ActiveMatch.AwayClub = WorldStateService.Universe.Clubs[m.Away]
			break
		end
	end

	if not ActiveMatch.HomeClub or not ActiveMatch.AwayClub then
		warn("❌ Match failed to initialize: Missing clubs.")
		return
	end

	ActiveMatch.IsRunning = true
	ActiveMatch.Minute = 0
	ActiveMatch.Score = {Home = 0, Away = 0}
	ActiveMatch.Phase = "Midfield" -- Started in Midfield Battle
	ActiveMatch.Possession = "Home"

	print("⚽ [KICKOFF] " .. ActiveMatch.HomeClub.Name .. " vs " .. ActiveMatch.AwayClub.Name)
	MatchUpdate:FireAllClients(ActiveMatch, "MatchStart", "KICKOFF!")

	-- Simulation Loop
	task.spawn(function()
		while ActiveMatch.Minute < 90 do
			task.wait(0.5) 
			ActiveMatch.Minute += 1

			if math.random(1, 4) == 1 then
				-- TACTICAL Tweak (Defensive = Less events, Attacking = More events)
				local homeTactic = ActiveMatch.HomeClub.Playstyle or "Balanced"
				local awayTactic = ActiveMatch.AwayClub.Playstyle or "Balanced"

				local eventChance = 25
				if homeTactic == "Attacking" or awayTactic == "Attacking" then eventChance = 35 end
				if homeTactic == "Defensive" or awayTactic == "Defensive" then eventChance = 15 end

				if math.random(1, 100) <= eventChance then
					ProcessTick()
				end
			end

			MatchUpdate:FireAllClients(ActiveMatch, "Tick", nil)
		end

		-- MATCH END
		ActiveMatch.IsRunning = false
		WorldStateService:RecordMatchResult(
			ActiveMatch.HomeClub.ID, 
			ActiveMatch.AwayClub.ID, 
			ActiveMatch.Score.Home, 
			ActiveMatch.Score.Away
		)

		-- Advance the week only if the player finished their game
		WorldStateService:AdvanceWeek()

		MatchUpdate:FireAllClients(ActiveMatch, "MatchEnd", "FULL TIME")
	end)
end)

return ActiveMatch


____________________________________________

Server

-- ================================================
-- SERVER ENTRY POINT (ServerScriptService - Script)
-- ================================================
-- The FIRST thing that runs. Boots all systems in order.
-- ================================================
local ReplicatedStorage  = game:GetService("ReplicatedStorage")
local ServerScriptService= game:GetService("ServerScriptService")
local Players            = game:GetService("Players")

warn("🚀 [SERVER] Football Manager Mode v3.0 — Booting...")

local function safeRequire(name)
	local module = script.Parent:FindFirstChild(name)
	if not module then
		warn("❌ [SERVER] " .. name .. " is MISSING in " .. script.Parent.Name .. "!")
		return nil
	end
	if not module:IsA("ModuleScript") then
		warn("❌ [SERVER] " .. name .. " is NOT a ModuleScript! (It is a " .. module.ClassName .. ")")
		return nil
	end
	return require(module)
end

local WorldStateService = safeRequire("WorldStateService")
local DataManager       = safeRequire("DataManager")
local LeagueService     = safeRequire("LeagueService")
local MatchService      = safeRequire("MatchService")
local AdminService      = safeRequire("AdminService") -- [NEW]
local ScoutService      = safeRequire("ScoutService") -- [NEW]

-- ================================================
-- 2. WORLD CLOCK
-- ================================================
local WEEK_DURATION    = 600  -- 10 minutes per in-game week (adjust for testing)
local IS_ADVANCING     = false

local function AdvanceWeek()
	if IS_ADVANCING then return end
	IS_ADVANCING = true

	local week   = WorldStateService.Universe.Week
	local season = WorldStateService.Universe.Season

	warn("📅 [CLOCK] Advancing to Week " .. week .. " Season " .. season)

	-- 1. Simulate all global matches (skip active player match)
	WorldStateService:SimulateGlobalMatches()

	-- 2. AI transfers
	WorldStateService:ProcessAITransfers()

	-- 3. Player development
	WorldStateService:UpdatePlayerDevelopment()

	-- 4. Advance week counter
	WorldStateService.Universe.Week += 1
	if WorldStateService.Universe.Week > 38 then
		WorldStateService.Universe.Week   = 1
		WorldStateService.Universe.Season += 1
		WorldStateService:PushNews("🏆 Season " .. season .. " is over! Season " .. season+1 .. " begins!")
		-- Process promotion/relegation at season end
		WorldStateService:ProcessPromotionRelegation()
	end

	IS_ADVANCING = false
end

-- Start world tick loop
task.spawn(function()
	task.wait(10) -- Small initial delay to let everything load
	while true do
		task.wait(WEEK_DURATION)
		AdvanceWeek()
	end
end)

-- 🚀 MISSION UPDATE LOOP (Fast 1s tick)
task.spawn(function()
	while true do
		task.wait(1)
		for _, player in ipairs(Players:GetPlayers()) do
			local profile = WorldStateService.Universe.Clubs[player.UserId] -- ID-based club ref
			if profile then
				ScoutService:Update(player.UserId)
			end
		end
	end
end)

-- ================================================
-- 📡 NETWORKING (Scout Requests)
-- ================================================
local function initNetworking()
	local remotes = ReplicatedStorage:WaitForChild("Remotes")
	local sendScout = remotes:WaitForChild("SendScout", 5) 

	if sendScout then
		sendScout.OnServerInvoke = function(player, missionType)
			return ScoutService:StartMission(player.UserId, missionType)
		end
	end
end
task.spawn(initNetworking)

-- ================================================
-- 3. PLAYER LIFECYCLE
-- ================================================
Players.PlayerAdded:Connect(function(player)
	DataManager:LoadData(player)
end)

Players.PlayerRemoving:Connect(function(player)
	DataManager:SaveData(player)
end)

-- Handle players who joined before the connection fired (Studio testing)
for _, player in ipairs(Players:GetPlayers()) do
	task.spawn(function() DataManager:LoadData(player) end)
end

-- ================================================
-- 4. AUTO-SAVE (every 60 seconds)
-- ================================================
task.spawn(function()
	while true do
		task.wait(60)
		for _, player in ipairs(Players:GetPlayers()) do
			DataManager:SaveData(player)
		end
	end
end)

warn("✅ [SERVER] All systems online. World Week " .. WorldStateService.Universe.Week .. " | Season " .. WorldStateService.Universe.Season)

_________________________________________

ReplicatedStorage

StatCalculator:

local StatCalculator = {}

-- Updated EA FC Weights (Total = 1.0 / 100%)
local STAT_WEIGHTS = {
	["Pace"] = 0.20,
	["Shooting"] = 0.20,
	["Passing"] = 0.15,
	["Dribbling"] = 0.15,
	["Physical"] = 0.15,
	["Defending"] = 0.15
}

function StatCalculator:CalculateOVR(stats)
	local total = 0
	for statName, weight in pairs(STAT_WEIGHTS) do
		-- Failsafe: if a stat is missing, default to 60
		total = total + (stats[statName] or 60) * weight
	end
	return math.floor(total + 0.5) -- Round to nearest whole number
end

return StatCalculator

______________________________________________

WorldGenerator

local WorldGenerator = {}
math.randomseed(os.time())

-- Name syllable tables by region
local FIRST = {
	"Mar","Car","Luc","And","Ser","Fer","Ric","Raf","Ben","Dav",
	"Jav","Ale","Tia","Emi","Rod","Cos","Fab","Vin","Gel","Nab",
	"Kal","Zel","Dre","Omar","Kei","Taj","Yao","Ibr","Sal","Mus"
}
local LAST = {
	"inho","eiro","andez","silva","ramos","ista","oga","etti",
	"ini","aro","eda","oma","aki","eze","diop","mane","kofi",
	"ovic","adic","berg","sson","qvist","ez","ano","ino","ello"
}
local CLUB_PREFIX = {
	"FC","AFC","SC","United","City","Athletic","Sporting","Real",
	"Racing","Dynamo","Viking","Titan","Iron","Storm","Phoenix","Nova"
}
local CLUB_SUFFIX = {
	"Vega","Torino","Aldea","Strom","Valdor","Krest","Mirova",
	"Castel","Ferron","Nordvik","Elara","Zenit","Marova","Brindal"
}
local TACTICS = {"Possession","Counter-Attack","High Press","Low Block","Direct Play"}
local POSITIONS = {"ST","LW","RW","CAM","CM","CDM","LB","RB","CB","GK"}

local POS_WEIGHTS = {
	ST=1, LW=1, RW=1, CAM=1, CM=2, CDM=1, LB=1, RB=1, CB=2, GK=1
}

-- Generate a realistic fictional name
function WorldGenerator:GenerateName()
	local first = FIRST[math.random(#FIRST)]
	local last = LAST[math.random(#LAST)]
	return first .. last
end

-- Generate a club name
function WorldGenerator:GenerateClubName(usedNames)
	local attempts = 0
	local name
	repeat
		local pre = CLUB_PREFIX[math.random(#CLUB_PREFIX)]
		local suf = CLUB_SUFFIX[math.random(#CLUB_SUFFIX)]
		name = pre .. " " .. suf
		attempts += 1
	until not usedNames[name] or attempts > 50
	usedNames[name] = true
	return name
end

-- Generate stats based on OVR and position
function WorldGenerator:GenerateStats(ovr, position)
	local base = ovr - 5
	local stats = {
		Shooting = math.random(base - 5, base + 5),
		Passing = math.random(base - 5, base + 5),
		Dribbling = math.random(base - 5, base + 5),
		Defending = math.random(base - 5, base + 5),
		Pace = math.random(base - 5, base + 5),
		Physical = math.random(base - 5, base + 5),
	}

	-- Position stat boosts
	if position == "ST" then stats.Shooting += 10 stats.Pace += 5 end
	if position == "CB" or position == "CDM" then stats.Defending += 12 stats.Physical += 5 end
	if position == "GK" then stats.Defending += 15 stats.Physical += 8 end
	if position == "CAM" or position == "LW" or position == "RW" then stats.Dribbling += 8 stats.Passing += 6 end
	if position == "CM" then stats.Passing += 8 stats.Physical += 4 end

	-- Clamp all stats
	for k, v in pairs(stats) do
		stats[k] = math.clamp(v, 40, 99)
	end

	return stats
end

-- Calculate OVR from stats (reusing StatCalculator logic)
function WorldGenerator:CalcOVR(stats)
	local weights = {Pace=0.20, Shooting=0.20, Passing=0.15, Dribbling=0.15, Physical=0.15, Defending=0.15}
	local total = 0
	for k, w in pairs(weights) do total += (stats[k] or 60) * w end
	return math.floor(total + 0.5)
end

-- Generate a single player
function WorldGenerator:GeneratePlayer(club, tier)
	local ovrRange = {
		top = {75, 90},
		mid = {62, 76},
		low = {50, 65}
	}

	-- Pick a position weighted properly
	local posPool = {}
	for pos, weight in pairs(POS_WEIGHTS) do
		for _ = 1, weight do table.insert(posPool, pos) end
	end
	local position = posPool[math.random(#posPool)]

	local range = ovrRange[tier]
	local ovr = math.random(range[1], range[2])
	local age = math.random(17, 33)
	local potential = math.min(99, ovr + math.random(0, math.max(0, 30 - (age - 18))))

	local stats = self:GenerateStats(ovr, position)
	ovr = self:CalcOVR(stats)

	local value = math.floor((ovr ^ 2) * 800 * (1 + (potential - ovr) * 0.02) * ((35 - age) * 0.05))
	value = math.max(value, 50000)

	return {
		Name = self:GenerateName(),
		Age = age,
		Position = position,
		OVR = ovr,
		Potential = potential,
		Stats = stats,
		Club = club or "Free Agent",
		Value = value,
		IsRetired = false
	}
end

-- Generate a Free Agent
function WorldGenerator:GenerateFreeAgent()
	local age = math.random(30, 38)
	local posPool = {}
	for pos, weight in pairs(POS_WEIGHTS) do
		for _ = 1, weight do table.insert(posPool, pos) end
	end
	local position = posPool[math.random(#posPool)]
	local ovr = math.random(55, 80)
	local potential = math.min(ovr + math.random(0, 3), 85)
	local stats = self:GenerateStats(ovr, position)
	ovr = self:CalcOVR(stats)

	return {
		Name = self:GenerateName(),
		Age = age,
		Position = position,
		OVR = ovr,
		Potential = potential,
		Stats = stats,
		Club = "Free Agent",
		Value = math.random(10000, 200000),
		IsRetired = false
	}
end

-- Generate an entire club
function WorldGenerator:GenerateClub(name, tier)
	local budgets = {top = 10000000, mid = 3000000, low = 800000}
	local squad = {}

	-- Generate 18 players (11 starters + 7 bench)
	for _ = 1, 18 do
		table.insert(squad, self:GeneratePlayer(name, tier))
	end

	-- Calculate club OVR
	local totalOVR = 0
	for _, p in ipairs(squad) do totalOVR += p.OVR end
	local clubOVR = math.floor(totalOVR / #squad)

	return {
		Name = name,
		Tier = tier,
		Tactic = TACTICS[math.random(#TACTICS)],
		Budget = budgets[tier],
		OVR = clubOVR,
		Squad = squad
	}
end

-- Generate the entire world
function WorldGenerator:GenerateWorld()
	local usedNames = {}
	local clubs = {}

	-- 6 top, 6 mid, 6 low tier clubs
	local tiers = {
		{tier = "top", count = 6},
		{tier = "mid", count = 6},
		{tier = "low", count = 6}
	}

	for _, t in ipairs(tiers) do
		for _ = 1, t.count do
			local name = self:GenerateClubName(usedNames)
			table.insert(clubs, self:GenerateClub(name, t.tier))
		end
	end

	-- Generate 20 free agents
	local freeAgents = {}
	for _ = 1, 20 do
		table.insert(freeAgents, self:GenerateFreeAgent())
	end

	return {Clubs = clubs, FreeAgents = freeAgents}
end

return WorldGenerator


Now ill get the summary from chatgpt
