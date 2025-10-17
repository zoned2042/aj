-- Vault Autojoiner (Wave/Volcano compatible)
-- UI: Running/Stop buttons, Auto toggle, Invite button (no Lock)
-- Pulls JobIds from local API: /next and /status
-- Load from executor: loadstring(game:HttpGet("http://127.0.0.1:5000/ui.lua"))()

local HttpService = game:GetService("HttpService")
local TeleportService = game:GetService("TeleportService")
local Players = game:GetService("Players")

local req = (syn and syn.request) or (http and http.request) or http_request or request or (fluxus and fluxus.request)
-- safe HTTP wrapper: opens UI even if request() isn't available; GET uses HttpGet fallback
local function http_call(params)
  if req then
    return req(params)
  else
    if (params.Method == "GET" or not params.Method) and params.Url then
      local ok, body = pcall(function() return game:HttpGet(params.Url) end)
      if ok then return { StatusCode = 200, Body = body } end
    end
    return { StatusCode = 0, Body = "" }
  end
end

local API = "http://127.0.0.1:5000"
local TITLE = "Vault Autojoiner"
local DISCORD = "https://discord.gg/cq29atkGA6" -- replace with your invite if desired

local state = {
  running = false,
  auto = false,
  lastJob = "",
  blacklist = {},
  min_money_raw = 0,
}

local _min_last = 0

-- Forward declarations for cross-references used before definitions
local showJoinInfo
local inBlacklist
local blacklistOn = true

local function jsonDecode(s)
  local ok, data = pcall(function() return HttpService:JSONDecode(s) end)
  if ok then return data end
  return nil
end

-- Normalize and validate a Roblox JobId string
local function normalizeJobId(id)
  if not id then return nil end
  local s = tostring(id)
  -- strip whitespace and quotes
  s = s:gsub("[\r\n\t]", " "):gsub("%s+", ""):gsub("^[\'\"]+", ""):gsub("[\'\"]+$", "")
  if #s < 32 or #s > 64 then return nil end
  if not s:match("^[0-9a-fA-F%-]+$") then return nil end
  if not s:find("%-") then return nil end
  return s
end

-- Attempt to parse placeId and jobId from a join_script string
local function parseJoinScript(js)
  if type(js) ~= "string" or #js == 0 then return nil, nil end
  -- Pattern: TeleportService:TeleportToPlaceInstance(123456, "guid")
  local pid, jid = js:match("TeleportToPlaceInstance%(%s*(%d+)%s*,%s*[%"']([0-9A-Fa-f%-]+)[%"']")
  if not pid or not jid then
    -- Fallback: placeId=12345 ... jobId="..."
    pid, jid = js:match("placeId%s*=%s*(%d+).-[jJ]ob[Ii]d%s*=%s*[%"']([0-9A-Fa-f%-]+)[%"']")
  end
  if not pid or not jid then
    -- Last resort: a number then a GUID-like token in quotes
    local p2, j2 = js:match("(%d+).-[%"']([0-9A-Fa-f%-]+)[%"']")
    if p2 and j2 then pid, jid = p2, j2 end
  end
  local njid = normalizeJobId(jid)
  local npid = pid and tonumber(pid) or nil
  return njid, npid
end

local function pollNext()
  -- include blacklist and money filter in POST
  local bl = {}
  for name,on in pairs(state.blacklist) do if on then table.insert(bl, name) end end
  local body = HttpService:JSONEncode({ blacklist = bl, min_money = (tonumber(state.min_money_raw) or 0) * 1000000 })
  local ok, res = pcall(function() return http_call({Url = API.."/next", Method = "POST", Headers = { ["Content-Type"] = "application/json" }, Body = body }) end)
  if not ok or not res then return nil end
  if res.StatusCode ~= 200 and res.StatusCode ~= 201 then
    -- try text fallback endpoint if available
    local ok2, res2 = pcall(function() return http_call({Url = API.."/next_text", Method = "GET"}) end)
    if ok2 and res2 and res2.StatusCode == 200 and res2.Body and #res2.Body > 5 then
      return normalizeJobId(tostring(res2.Body)), nil
    end
    return nil
  end
  local data = jsonDecode(res.Body or "")
  if data and data.ok and data.server then
    local jid, pid = nil, nil
    if data.server.join_script then
      jid, pid = parseJoinScript(data.server.join_script)
    end
    if not jid and data.server.job_id then
      jid = normalizeJobId(tostring(data.server.job_id))
    end
    if not pid and data.server.place_id then
      pid = tonumber(data.server.place_id)
    end
    if jid then return jid, pid end
  end
  return nil
end

local function turboNextText()
  local ok, res = pcall(function() return http_call({Url = API.."/next_text", Method = "GET"}) end)
  if ok and res and res.StatusCode == 200 and res.Body and #res.Body > 5 then
    return normalizeJobId(tostring(res.Body))
  end
  return nil
end

local function teleport(jobId, placeId)
  jobId = normalizeJobId(jobId)
  if not jobId or #jobId == 0 then return end
  local pid = placeId or game.PlaceId
  TeleportService:TeleportToPlaceInstance(pid, jobId, Players.LocalPlayer)
end

-- Turbo path that respects current blacklist: if any blacklist is on, prefer POST /next
local function turboGetJobId()
  local hasBl=false
  for _,on in pairs(state.blacklist) do if on then hasBl=true break end end
  local minOn = (tonumber(state.min_money_raw) or 0) > 0
  -- Prefer JSON /next (gives join_script/placeId) for accuracy
  local bl={}; for k,v in pairs(state.blacklist) do if v then table.insert(bl,k) end end
  local body = HttpService:JSONEncode({ blacklist = bl, min_money = (tonumber(state.min_money_raw) or 0) * 1000000 })
  local ok,res = pcall(function()
    return http_call({ Url = API.."/next", Method = "POST", Headers = { ["Content-Type"] = "application/json" }, Body = body })
  end)
  if ok and res and (res.StatusCode==200 or res.StatusCode==201) and res.Body then
    local d=jsonDecode(res.Body)
    if d and d.ok and d.server then
      local jid,pid = parseJoinScript(d.server.join_script or "")
      if not jid and d.server.job_id then jid = normalizeJobId(tostring(d.server.job_id)) end
      if not pid and d.server.place_id then pid = tonumber(d.server.place_id) end
      if jid then return jid, pid end
    end
  end
  -- Fallback: choose highest value from /servers
  local ok2, res2 = pcall(function() return http_call({ Url = API.."/servers", Method = "GET" }) end)
  if ok2 and res2 and res2.StatusCode == 200 and res2.Body then
    local d2 = jsonDecode(res2.Body)
    if d2 and d2.servers and type(d2.servers) == "table" then
      local best = nil
      for _, it in ipairs(d2.servers) do
        local nm = tostring(it.pet_name or "")
        local val = tonumber(it.value or 0) or 0
        if not (blacklistOn and inBlacklist(nm)) and (not string.find(string.lower(nm), "brainrot notify")) then
          if not minOn or val >= (state.min_money_raw * 1e6) then
            if (not best) or (val > (tonumber(best.value or 0) or 0)) then best = it end
          end
        end
      end
      if best then
        local jid, pid = nil, nil
        if best.join_script then jid, pid = parseJoinScript(best.join_script) end
        if not jid and best.job_id then jid = normalizeJobId(tostring(best.job_id)) end
        if not pid and best.place_id then pid = tonumber(best.place_id) end
        if jid then return jid, pid end
      end
    end
  end
  -- Last resort: text endpoint then pollNext
  local tjid = turboNextText()
  if tjid then return tjid, nil end
  return pollNext()
end

-- Minimal UI (executor-native), avoid heavy libraries; pcall sections for robustness
-- Keybinds: [T]=Auto toggle, [H]=Invi toggle, [I]=Copy invite, [J]=Quick join
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "VaultJoinerUI"
ScreenGui.ResetOnSpawn = false
ScreenGui.IgnoreGuiInset = true
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling

local function protect_gui(gui)
  pcall(function()
    if syn and syn.protect_gui then syn.protect_gui(gui) end
  end)
  local parent = (gethui and gethui())
    or (get_hidden_gui and get_hidden_gui())
    or (game:FindFirstChildOfClass("CoreGui"))
    or game:GetService("Players").LocalPlayer:WaitForChild("PlayerGui")
    or game:GetService("StarterGui")
  gui.Parent = parent
end

local green = Color3.fromRGB(24,164,99)
local red = Color3.fromRGB(196,62,62)
local gray = Color3.fromRGB(28,28,28)
local gray2 = Color3.fromRGB(40,40,40)
local gray3 = Color3.fromRGB(52,52,52)
local accent = Color3.fromRGB(30,180,140)
local orange = Color3.fromRGB(214,130,46)

-- Root floating panel like screenshot
local Root = Instance.new("Frame")
Root.Size = UDim2.new(0, 820, 0, 520)
Root.Position = UDim2.new(0.5, -410, 0.5, -260)
Root.BackgroundColor3 = gray
Root.BorderSizePixel = 0
Root.Active = true
Root.Draggable = true
Root.Parent = ScreenGui

local Header = Instance.new("Frame")
Header.Size = UDim2.new(1, 0, 0, 44)
Header.BackgroundColor3 = gray2
Header.BorderSizePixel = 0
Header.Parent = Root

local Title = Instance.new("TextLabel")
Title.BackgroundTransparency = 1
Title.Position = UDim2.new(0, 14, 0, 8)
Title.Size = UDim2.new(1, -28, 0, 28)
Title.Font = Enum.Font.GothamBold
Title.TextSize = 18
Title.TextXAlignment = Enum.TextXAlignment.Left
Title.TextColor3 = Color3.fromRGB(230,230,230)
Title.Text = "Auto Joiner | Vault"
Title.Parent = Header

local RunningDot = Instance.new("TextLabel")
RunningDot.BackgroundTransparency = 1
RunningDot.Size = UDim2.new(0, 120, 1, 0)
RunningDot.Position = UDim2.new(1, -130, 0, 0)
RunningDot.Font = Enum.Font.Gotham
RunningDot.TextSize = 14
RunningDot.TextXAlignment = Enum.TextXAlignment.Right
RunningDot.TextColor3 = Color3.fromRGB(130,130,130)
RunningDot.Text = "+ IDLE"
RunningDot.Parent = Header

local Body = Instance.new("Frame")
Body.BackgroundColor3 = gray
Body.BorderSizePixel = 0
Body.Position = UDim2.new(0, 0, 0, 44)
Body.Size = UDim2.new(1, 0, 1, -44)
Body.Parent = Root

local Left = Instance.new("Frame")
Left.BackgroundColor3 = gray
Left.BorderSizePixel = 0
Left.Position = UDim2.new(0, 0, 0, 0)
Left.Size = UDim2.new(0, 270, 1, 0)
Left.Parent = Body

local Right = Instance.new("Frame")
Right.BackgroundColor3 = gray
Right.BorderSizePixel = 0
Right.Position = UDim2.new(0, 270, 0, 0)
Right.Size = UDim2.new(1, -270, 1, 0)
Right.Parent = Body

local function makeBtn(parent, text, bg, pos)
  local b = Instance.new("TextButton")
  b.Size = UDim2.new(0, 120, 0, 36)
  b.Position = pos
  b.BackgroundColor3 = bg
  b.BorderSizePixel = 0
  b.TextColor3 = Color3.fromRGB(255,255,255)
  b.Font = Enum.Font.GothamBold
  b.TextSize = 14
  b.Text = text
  b.AutoButtonColor = true
  b.Parent = parent
  return b
end

local runningBtn = makeBtn(Left, "START", green, UDim2.new(0, 16, 0, 16))
local stopBtn = makeBtn(Left, "STOP", red, UDim2.new(0, 146, 0, 16))
local turboBtn = makeBtn(Left, "TURBO [J]", Color3.fromRGB(200,120,60), UDim2.new(0, 16, 0, 170))

local autoLabel = Instance.new("TextLabel")
autoLabel.BackgroundTransparency = 1
autoLabel.Position = UDim2.new(0, 16, 0, 66)
autoLabel.Size = UDim2.new(0, 240, 0, 24)
autoLabel.Font = Enum.Font.Gotham
autoLabel.TextSize = 14
autoLabel.TextColor3 = Color3.fromRGB(210,210,210)
autoLabel.TextXAlignment = Enum.TextXAlignment.Left
autoLabel.Text = "AUTO JOIN: OFF"
autoLabel.Parent = Left

-- Money filter label and controls
local minLabel = Instance.new("TextLabel")
minLabel.BackgroundTransparency = 1
minLabel.Position = UDim2.new(0, 16, 0, 86)
minLabel.Size = UDim2.new(0, 240, 0, 20)
minLabel.Font = Enum.Font.Gotham
minLabel.TextSize = 14
minLabel.TextColor3 = Color3.fromRGB(200,200,200)
minLabel.TextXAlignment = Enum.TextXAlignment.Left
minLabel.Text = "MIN: 0M"
minLabel.Parent = Left

local mfToggle = Instance.new("TextButton")
mfToggle.Size = UDim2.new(0, 120, 0, 22)
mfToggle.Position = UDim2.new(0, 16, 0, 110)
mfToggle.BackgroundColor3 = Color3.fromRGB(90,90,90)
mfToggle.BorderSizePixel = 0
mfToggle.TextColor3 = Color3.fromRGB(255,255,255)
mfToggle.Font = Enum.Font.GothamBold
mfToggle.TextSize = 13
mfToggle.Text = "FILTER: OFF"
mfToggle.Parent = Left

local minusBtn = Instance.new("TextButton")
minusBtn.Size = UDim2.new(0, 28, 0, 22)
minusBtn.Position = UDim2.new(0, 182, 0, 86)
minusBtn.BackgroundColor3 = Color3.fromRGB(120,120,120)
minusBtn.BorderSizePixel = 0
minusBtn.TextColor3 = Color3.fromRGB(255,255,255)
minusBtn.Font = Enum.Font.GothamBold
minusBtn.TextSize = 14
minusBtn.Text = "-"
minusBtn.Parent = Left

local plusBtn = Instance.new("TextButton")
plusBtn.Size = UDim2.new(0, 28, 0, 22)
plusBtn.Position = UDim2.new(0, 214, 0, 86)
plusBtn.BackgroundColor3 = Color3.fromRGB(120,120,120)
plusBtn.BorderSizePixel = 0
plusBtn.TextColor3 = Color3.fromRGB(255,255,255)
plusBtn.Font = Enum.Font.GothamBold
plusBtn.TextSize = 14
plusBtn.Text = "+"
plusBtn.Parent = Left

local blacklistLabel = Instance.new("TextLabel")
blacklistLabel.BackgroundTransparency = 1
blacklistLabel.Position = UDim2.new(0, 16, 0, 102)
blacklistLabel.Size = UDim2.new(0, 240, 0, 20)
blacklistLabel.Font = Enum.Font.Gotham
blacklistLabel.TextSize = 14
blacklistLabel.TextColor3 = Color3.fromRGB(200,200,200)
blacklistLabel.TextXAlignment = Enum.TextXAlignment.Left
blacklistLabel.Text = "BLACKLIST"
blacklistLabel.Parent = Left

local blToggle = makeBtn(Left, "BLACKLIST ON", green, UDim2.new(0, 16, 0, 126))

local inviBtn = makeBtn(Left, "INVI: OFF [H]", Color3.fromRGB(68,114,196), UDim2.new(0, 16, 0, 214))

-- Brainrot blacklist box
local blListLabel = Instance.new("TextLabel")
blListLabel.BackgroundTransparency = 1
blListLabel.Position = UDim2.new(0, 16, 0, 254)
blListLabel.Size = UDim2.new(0, 240, 0, 20)
blListLabel.Font = Enum.Font.Gotham
blListLabel.TextSize = 14
blListLabel.TextColor3 = Color3.fromRGB(200,200,200)
blListLabel.TextXAlignment = Enum.TextXAlignment.Left
blListLabel.Text = "BRAINROT BLACKLIST"
blListLabel.Parent = Left

-- Input box + ADD + PURGE
local blInput = Instance.new("TextBox")
blInput.Position = UDim2.new(0, 16, 0, 278)
blInput.Size = UDim2.new(0, 160, 0, 28)
blInput.BackgroundColor3 = gray3
blInput.BorderSizePixel = 0
blInput.ClearTextOnFocus = false
blInput.PlaceholderText = "add term..."
blInput.Text = ""
blInput.TextColor3 = Color3.fromRGB(230,230,230)
blInput.Font = Enum.Font.Gotham
blInput.TextSize = 14
blInput.Parent = Left

local blAddBtn = makeBtn(Left, "ADD", Color3.fromRGB(90,140,220), UDim2.new(0, 182, 0, 278))
blAddBtn.Size = UDim2.new(0, 74, 0, 28)

local purgeBtn = makeBtn(Left, "PURGE BLACKLISTED", Color3.fromRGB(120,60,60), UDim2.new(0, 16, 0, 318))
purgeBtn.Size = UDim2.new(0, 240, 0, 28)

local blChipScroll = Instance.new("ScrollingFrame")
blChipScroll.Position = UDim2.new(0, 16, 0, 354)
blChipScroll.Size = UDim2.new(0, 240, 0, 120)
blChipScroll.BackgroundColor3 = gray3
blChipScroll.BorderSizePixel = 0
blChipScroll.ScrollBarThickness = 6
blChipScroll.CanvasSize = UDim2.new(0,0,0,0)
blChipScroll.Parent = Left
local blChipLayout = Instance.new("UIListLayout")
blChipLayout.Padding = UDim.new(0, 6)
blChipLayout.FillDirection = Enum.FillDirection.Vertical
blChipLayout.SortOrder = Enum.SortOrder.LayoutOrder
blChipLayout.Parent = blChipScroll

-- Simple tabs: Servers / Multi Pets
local tabBar = Instance.new("Frame")
tabBar.BackgroundColor3 = gray
tabBar.BorderSizePixel = 0
tabBar.Size = UDim2.new(1, -16, 0, 28)
tabBar.Position = UDim2.new(0, 8, 0, 8)
tabBar.Parent = Right

local function makeTab(txt, x)
  local b = Instance.new("TextButton")
  b.Size = UDim2.new(0, 110, 0, 28)
  b.Position = UDim2.new(0, x, 0, 0)
  b.BackgroundColor3 = gray2
  b.BorderSizePixel = 0
  b.TextColor3 = Color3.fromRGB(230,230,230)
  b.Font = Enum.Font.GothamBold
  b.TextSize = 14
  b.Text = txt
  b.AutoButtonColor = true
  b.Parent = tabBar
  return b
end
local tabBtnServers = makeTab("SERVERS", 0)
local tabBtnMulti = makeTab("MULTI PETS", 120)
local currentTab = "servers"

-- Servers header
local rowHeader = Instance.new("Frame")
rowHeader.BackgroundColor3 = gray2
rowHeader.BorderSizePixel = 0
rowHeader.Size = UDim2.new(1, -16, 0, 34)
rowHeader.Position = UDim2.new(0, 8, 0, 50)
rowHeader.Parent = Right

local function headerText(txt, x)
  local l = Instance.new("TextLabel")
  l.BackgroundTransparency = 1
  l.Position = UDim2.new(0, x, 0, 7)
  l.Size = UDim2.new(0, 160, 0, 20)
  l.Font = Enum.Font.GothamBold
  l.TextSize = 14
  l.TextColor3 = Color3.fromRGB(220,220,220)
  l.TextXAlignment = Enum.TextXAlignment.Left
  l.Text = txt
  l.Parent = rowHeader
end
headerText("PET", 12)
headerText("MONEY/s", 350)
headerText("ACTION", 500)

-- Multi Pets boxed panel (hidden unless tab is active)
local mpBox = Instance.new("Frame")
mpBox.BackgroundColor3 = gray2
mpBox.BorderSizePixel = 0
mpBox.Position = UDim2.new(0, 8, 0, 50)
mpBox.Size = UDim2.new(1, -16, 0, 280)
mpBox.Visible = false
mpBox.Parent = Right

local mpHeader = Instance.new("TextLabel")
mpHeader.BackgroundTransparency = 1
mpHeader.Position = UDim2.new(0, 8, 0, 6)
mpHeader.Size = UDim2.new(1, -16, 0, 16)
mpHeader.Font = Enum.Font.GothamBold
mpHeader.TextSize = 14
mpHeader.TextXAlignment = Enum.TextXAlignment.Left
mpHeader.TextColor3 = Color3.fromRGB(210,210,210)
mpHeader.Text = "MULTI PET"
mpHeader.Parent = mpBox

local mpList = Instance.new("Frame")
mpList.BackgroundTransparency = 1
mpList.Position = UDim2.new(0, 8, 0, 26)
mpList.Size = UDim2.new(1, -16, 0, 64)
mpList.Parent = mpBox
local mpLayout = Instance.new("UIListLayout")
mpLayout.FillDirection = Enum.FillDirection.Vertical
mpLayout.SortOrder = Enum.SortOrder.LayoutOrder
mpLayout.Padding = UDim.new(0, 2)
mpLayout.Parent = mpList

local Scroller = Instance.new("ScrollingFrame")
Scroller.Position = UDim2.new(0, 8, 0, 84)
Scroller.Size = UDim2.new(1, -16, 1, -100)
Scroller.BackgroundTransparency = 1
Scroller.CanvasSize = UDim2.new(0, 0, 0, 0)
Scroller.ScrollBarThickness = 6
Scroller.Parent = Right

local ListLayout = Instance.new("UIListLayout")
ListLayout.Padding = UDim.new(0, 8)
ListLayout.Parent = Scroller

-- Toast label for feedback
local toast = Instance.new("TextLabel")
toast.Size = UDim2.new(1, 0, 0, 22)
toast.Position = UDim2.new(0, 0, 0, 0)
toast.BackgroundColor3 = Color3.fromRGB(20,20,20)
toast.BackgroundTransparency = 0.2
toast.Visible = false
toast.Text = "Invite copied to clipboard"
toast.TextColor3 = Color3.fromRGB(200,200,255)
toast.Font = Enum.Font.Gotham
toast.TextSize = 14
toast.Parent = ScreenGui

local function setRunning(on)
  state.running = on and true or false
  runningBtn.BackgroundColor3 = on and green or Color3.fromRGB(90,90,90)
  RunningDot.TextColor3 = on and green or Color3.fromRGB(130,130,130)
  RunningDot.Text = on and "+ RUNNING" or "+ IDLE"
end

local function setAuto(on)
  state.auto = on and true or false
  autoLabel.Text = on and "AUTO JOIN: ON" or "AUTO JOIN: OFF"
end

local function setMinRaw(m)
  local mnum = tonumber(m) or 0
  if mnum < 0 then mnum = 0 end
  state.min_money_raw = mnum
  minLabel.Text = "MIN: "..tostring(mnum).."M"
end

local function setFilterOn(on)
  if on then
    if (tonumber(state.min_money_raw) or 0) <= 0 then
      if _min_last <= 0 then _min_last = 5 end
      setMinRaw(_min_last)
    end
    mfToggle.Text = "FILTER: ON"
    mfToggle.BackgroundColor3 = green
  else
    _min_last = tonumber(state.min_money_raw) or 0
    setMinRaw(0)
    mfToggle.Text = "FILTER: OFF"
    mfToggle.BackgroundColor3 = Color3.fromRGB(90,90,90)
  end
end

local heartbeat
local _lastPost = 0
local _blVersion = 0
local _blJsonCache = "{}"
local function stop()
  setRunning(false)
  if heartbeat then heartbeat:Disconnect() heartbeat = nil end
end

local function start()
  if state.running then return end
  setRunning(true)
  heartbeat = game:GetService("RunService").Heartbeat:Connect(function()
    if not state.running then return end
    if state.auto then
      -- send blacklist-aware request with a small cooldown to avoid lag
      local nowt = tick()
  if nowt - _lastPost < 0.15 then return end
      _lastPost = nowt
      local bl = {}
      for name, on in pairs(state.blacklist) do if on then table.insert(bl, name) end end
      local body = HttpService:JSONEncode({ blacklist = bl, min_money = (tonumber(state.min_money_raw) or 0) * 1000000 })
      local ok, res = pcall(function()
        return http_call({Url = API.."/next", Method = "POST", Headers = { ["Content-Type"] = "application/json" }, Body = body })
      end)
      local jid, pid
      if ok and res and (res.StatusCode == 200 or res.StatusCode == 201) and res.Body then
        local data = jsonDecode(res.Body)
        if data and data.ok and data.server then
          if data.server.join_script then
            jid, pid = parseJoinScript(data.server.join_script)
          end
          if not jid and data.server.job_id then
            jid = normalizeJobId(tostring(data.server.job_id))
          end
          if not pid and data.server.place_id then
            pid = tonumber(data.server.place_id)
          end
        end
      end
      if jid and jid ~= state.lastJob then
        state.lastJob = jid
        showJoinInfo({ source = "Auto" }, jid, pid)
        teleport(jid, pid)
      end
    end
  end)
end

runningBtn.MouseButton1Click:Connect(function()
  setAuto(true)
  start()
end)

stopBtn.MouseButton1Click:Connect(function()
  setAuto(false)
  stop()
end)

turboBtn.MouseButton1Click:Connect(function()
  local jid, pid = turboGetJobId()
  if jid then
    showJoinInfo({ source = "Turbo" }, jid, pid)
    teleport(jid, pid)
  end
end)

-- Right header quick button: JOIN LATEST
local joinLatestBtnR = Instance.new("TextButton")
joinLatestBtnR.Size = UDim2.new(0, 120, 0, 28)
joinLatestBtnR.Position = UDim2.new(1, -136, 0, 8)
joinLatestBtnR.BackgroundColor3 = Color3.fromRGB(160,100,220)
joinLatestBtnR.BorderSizePixel = 0
joinLatestBtnR.TextColor3 = Color3.fromRGB(255,255,255)
joinLatestBtnR.Font = Enum.Font.GothamBold
joinLatestBtnR.TextSize = 14
joinLatestBtnR.Text = "JOIN LATEST"
joinLatestBtnR.Parent = Right
joinLatestBtnR.MouseButton1Click:Connect(function()
  local jid, pid = turboGetJobId()
  if jid then
    -- spam teleport a few times for reliability
    local tries = 3
    showJoinInfo({ source = "Join Latest" }, jid, pid)
    task.spawn(function()
      for i=1,tries do teleport(jid, pid) task.wait(0.15) end
    end)
  end
end)

local function setclipboard_any(text)
  local ok = false
  if setclipboard then pcall(function() setclipboard(text) end); ok = true end
  if (not ok) and toclipboard then pcall(function() toclipboard(text) end); ok = true end
  if (not ok) and set_clipboard then pcall(function() set_clipboard(text) end); ok = true end
  return ok
end

local function showToast(msg)
  toast.Text = msg
  toast.Visible = true
  task.delay(1.5, function() toast.Visible = false end)
end

-- Helpers to show which server we're joining
local function fmtMoney(v)
  local n = tonumber(v or 0) or 0
  if n <= 0 then return "--" end
  return "$"..string.format("%.1f", n/1e6).."M/s"
end

local function shortJob(j)
  j = tostring(j or "")
  if #j <= 8 then return j end
  return string.sub(j, 1, 8).."…"
end

local function showJoinInfo(meta, jid, pid)
  local name = meta and meta.pet_name or nil
  local val = meta and meta.value or nil
  local src = meta and meta.source or nil
  local money = fmtMoney(val)
  local parts = {}
  if name and #tostring(name) > 0 then table.insert(parts, tostring(name)) end
  if money ~= "--" then table.insert(parts, money) end
  local info = table.concat(parts, "  –  ")
  if #info == 0 then
    info = (pid and ("place "..tostring(pid)) or "")
    if #info > 0 then info = info .. " • " end
    info = info .. "job " .. shortJob(jid or "")
  end
  if src and #src > 0 then
    showToast("Joining: "..info.."  ("..src..")")
  else
    showToast("Joining: "..info)
  end
end

-- Invi (local base invisibility) toggle
local invi = { on = false, modified = {} }
local inviRadius = 80 -- studs around the player

local function _validPart(p, character)
  if not p or not p.Parent then return false end
  if not p:IsA("BasePart") then return false end
  if character and p:IsDescendantOf(character) then return false end
  return true
end

local function invi_apply()
  local plr = Players.LocalPlayer
  local char = plr and plr.Character
  local hrp = char and char:FindFirstChild("HumanoidRootPart")
  if not hrp then return end
  local params = OverlapParams.new()
  params.FilterType = Enum.RaycastFilterType.Exclude
  params.FilterDescendantsInstances = { char }
  local parts = workspace:GetPartBoundsInRadius(hrp.Position, inviRadius, params)
  for _, p in ipairs(parts) do
    if _validPart(p, char) then
      if p.LocalTransparencyModifier < 0.8 then
        p.LocalTransparencyModifier = 0.8
        invi.modified[p] = true
      end
    end
  end
end

local function invi_clear()
  for p, _ in pairs(invi.modified) do
    if typeof(p) == "Instance" and p.Parent and p:IsA("BasePart") then
  p.LocalTransparencyModifier = 0
    end
  end
  invi.modified = {}
end

local function toggleInvi()
  invi.on = not invi.on
  if invi.on then
    inviBtn.Text = "INVI: ON  [H]"
    inviBtn.BackgroundColor3 = Color3.fromRGB(90,160,230)
    showToast("Invi ON — nearby base hidden")
    invi_apply()
  else
    inviBtn.Text = "INVI: OFF [H]"
    inviBtn.BackgroundColor3 = Color3.fromRGB(68,114,196)
    invi_clear()
    showToast("Invi OFF — base restored")
  end
end

-- keep it fresh while ON: periodically apply to newly streamed parts
task.spawn(function()
  while true do
    task.wait(0.75)
    if invi.on then pcall(invi_apply) end
  end
end)

inviBtn.MouseButton1Click:Connect(toggleInvi)

-- Keybinds
local UIS = game:GetService("UserInputService")
UIS.InputBegan:Connect(function(input, gp)
  if gp then return end
  if input.KeyCode == Enum.KeyCode.T then
    setAuto(not state.auto)
  elseif input.KeyCode == Enum.KeyCode.H then
    toggleInvi()
  elseif input.KeyCode == Enum.KeyCode.Backquote then
    -- Hide/show entire UI with the backtick/tilde key
    ScreenGui.Enabled = not ScreenGui.Enabled
  elseif input.KeyCode == Enum.KeyCode.I then
    if setclipboard_any(DISCORD) then showToast("Invite copied to clipboard") else showToast("Copy failed; check executor perms") end
  elseif input.KeyCode == Enum.KeyCode.R then
    -- Config placeholder: future settings
  elseif input.KeyCode == Enum.KeyCode.J then
    local jid, pid = turboGetJobId()
    if jid then
      showJoinInfo({ source = "J Key" }, jid, pid)
      teleport(jid, pid)
    end
  elseif input.KeyCode == Enum.KeyCode.M then
    -- cycle 0 -> 5 -> 10 -> 20 -> 50 -> 0
    local cur = tonumber(state.min_money_raw) or 0
    local nextv = 0
    if cur < 5 then nextv = 5 elseif cur < 10 then nextv = 10 elseif cur < 20 then nextv = 20 elseif cur < 50 then nextv = 50 else nextv = 0 end
    setFilterOn(nextv > 0)
    setMinRaw(nextv)
    showToast("Money filter: "..tostring(state.min_money_raw).."M+")
  elseif input.KeyCode == Enum.KeyCode.LeftBracket then
    local v = (tonumber(state.min_money_raw) or 0) - 5
    if v < 0 then v = 0 end
    setFilterOn(v > 0)
    setMinRaw(v)
    showToast("Money filter: "..tostring(state.min_money_raw).."M+")
  elseif input.KeyCode == Enum.KeyCode.RightBracket then
    local v = (tonumber(state.min_money_raw) or 0) + 5
    setFilterOn(v > 0)
    setMinRaw(v)
    showToast("Money filter: "..tostring(state.min_money_raw).."M+")
  end
end)

-- blacklist feature (client-side)
local blacklistOn = true
local blacklist = {}
local function addBlacklist(name)
  if not name then return end
  name = string.lower((tostring(name) or ""):gsub("^%s+","" ):gsub("%s+$",""))
  if #name == 0 then return end
  blacklist[name] = true
end
local function inBlacklist(name)
  if not name then return false end
  return blacklist[string.lower(name)] == true
end
blToggle.MouseButton1Click:Connect(function()
  blacklistOn = not blacklistOn
  blToggle.Text = blacklistOn and "BLACKLIST ON" or "BLACKLIST OFF"
  blToggle.BackgroundColor3 = blacklistOn and green or Color3.fromRGB(120,120,120)
end)

-- Update local state.blacklist whenever user adds entries
local function rebuildBlacklist()
  state.blacklist = {}
  -- For simplicity we keep a set mirror; we'll only add via UI add button
  -- Existing entries live in the dictionary 'blacklist'
  for k, v in pairs(blacklist) do
    if v then state.blacklist[k] = true end
  end
  -- Rebuild cached JSON body for blacklist-aware /next and /purge
  local bl = {}
  for k,v in pairs(blacklist) do if v then table.insert(bl, k) end end
  _blJsonCache = HttpService:JSONEncode({ blacklist = bl })
end
blToggle.MouseButton1Click:Connect(function() rebuildBlacklist() end)

-- Render blacklist chips list
local function renderBlacklistChips()
  for _, c in ipairs(blChipScroll:GetChildren()) do
    if c:IsA("Frame") then c:Destroy() end
  end
  local total = 0
  for name, on in pairs(blacklist) do
    if on then
      total = total + 1
      local chip = Instance.new("Frame")
      chip.BackgroundColor3 = Color3.fromRGB(60,60,60)
      chip.BorderSizePixel = 0
      chip.Size = UDim2.new(1, -6, 0, 24)
      chip.Parent = blChipScroll

      local lbl = Instance.new("TextLabel")
      lbl.BackgroundTransparency = 1
      lbl.Position = UDim2.new(0, 8, 0, 4)
      lbl.Size = UDim2.new(1, -60, 0, 16)
      lbl.Font = Enum.Font.Gotham
      lbl.TextSize = 13
      lbl.TextXAlignment = Enum.TextXAlignment.Left
      lbl.TextColor3 = Color3.fromRGB(220,220,220)
      lbl.Text = name
      lbl.Parent = chip

      local xBtn = Instance.new("TextButton")
      xBtn.Size = UDim2.new(0, 24, 0, 20)
      xBtn.Position = UDim2.new(1, -30, 0, 2)
      xBtn.BackgroundColor3 = Color3.fromRGB(120,60,60)
      xBtn.BorderSizePixel = 0
      xBtn.TextColor3 = Color3.fromRGB(255,255,255)
      xBtn.Font = Enum.Font.GothamBold
      xBtn.TextSize = 14
      xBtn.Text = "X"
      xBtn.Parent = chip
      xBtn.MouseButton1Click:Connect(function()
        blacklist[name] = nil
        rebuildBlacklist()
        _blVersion = _blVersion + 1
        renderBlacklistChips()
        -- auto-purge removed terms
        pcall(function() return http_call({ Url = API.."/purge", Method = "POST", Headers = { ["Content-Type"] = "application/json" }, Body = _blJsonCache }) end)
      end)
    end
  end
  blChipScroll.CanvasSize = UDim2.new(0,0,0, total * 28)
end

-- Seed defaults to filter random/notify posts
addBlacklist("brainrot")
addBlacklist("brain rot")
addBlacklist("brain-rot")
addBlacklist("brainrot notify")
addBlacklist("ajjans hub")
addBlacklist("high value multi-pet")
rebuildBlacklist()
renderBlacklistChips()

-- Add/Purge controls
blAddBtn.MouseButton1Click:Connect(function()
  local term = (tostring(blInput.Text or "") or ""):gsub("^%s+",""):gsub("%s+$","")
  if #term > 0 then
    addBlacklist(term)
    blInput.Text = ""
    rebuildBlacklist()
    renderBlacklistChips()
    -- purge on add
    pcall(function() return http_call({ Url = API.."/purge", Method = "POST", Headers = { ["Content-Type"] = "application/json" }, Body = _blJsonCache }) end)
  end
end)

purgeBtn.MouseButton1Click:Connect(function()
  rebuildBlacklist()
  pcall(function() return http_call({ Url = API.."/purge", Method = "POST", Headers = { ["Content-Type"] = "application/json" }, Body = _blJsonCache }) end)
end)

-- Wire Money Filter buttons
mfToggle.MouseButton1Click:Connect(function()
  local on = (tonumber(state.min_money_raw) or 0) <= 0
  setFilterOn(on)
  showToast(on and ("Money filter: "..tostring(state.min_money_raw).."M+") or "Money filter: OFF")
end)
minusBtn.MouseButton1Click:Connect(function()
  local v = (tonumber(state.min_money_raw) or 0) - 5
  if v < 0 then v = 0 end
  setMinRaw(v)
  if v <= 0 then setFilterOn(false) else setFilterOn(true) end
  showToast("Money filter: "..tostring(state.min_money_raw).."M+")
end)
plusBtn.MouseButton1Click:Connect(function()
  local v = (tonumber(state.min_money_raw) or 0) + 5
  setMinRaw(v)
  if v <= 0 then setFilterOn(false) else setFilterOn(true) end
  showToast("Money filter: "..tostring(state.min_money_raw).."M+")
end)

-- Make rows from /servers queue info
local function makeRow(info)
  local row = Instance.new("Frame")
  row.BackgroundColor3 = gray2
  row.BorderSizePixel = 0
  row.Size = UDim2.new(1, -10, 0, 40)

  local pet = Instance.new("TextLabel")
  pet.BackgroundTransparency = 1
  pet.Position = UDim2.new(0, 10, 0, 10)
  pet.Size = UDim2.new(0, 320, 0, 20)
  pet.Font = Enum.Font.Gotham
  pet.TextSize = 14
  pet.TextColor3 = Color3.fromRGB(230,230,230)
  pet.TextXAlignment = Enum.TextXAlignment.Left
  pet.Text = tostring(info.pet_name or "Unknown")
  pet.Parent = row

  -- Players column removed per request

  local money = Instance.new("TextLabel")
  money.BackgroundTransparency = 1
  money.Position = UDim2.new(0, 350, 0, 10)
  money.Size = UDim2.new(0, 100, 0, 20)
  money.Font = Enum.Font.GothamBold
  money.TextSize = 14
  money.TextColor3 = Color3.fromRGB(50,220,140)
  local v = tonumber(info.value or 0) or 0
  if v > 0 then
    money.Text = string.format("$%.1fM/s", v/1e6)
  else
    money.Text = "--"
  end
  money.Parent = row

  local join = Instance.new("TextButton")
  join.Size = UDim2.new(0, 90, 0, 28)
  join.Position = UDim2.new(0, 500, 0, 6)
  join.BackgroundColor3 = Color3.fromRGB(200,60,60)
  join.BorderSizePixel = 0
  join.TextColor3 = Color3.fromRGB(255,255,255)
  join.Font = Enum.Font.GothamBold
  join.TextSize = 14
  join.Text = "JOIN"
  join.Parent = row
  join.MouseButton1Click:Connect(function()
    local jid, pid = nil, nil
    if info.join_script then jid, pid = parseJoinScript(info.join_script) end
    if not jid and info.job_id then jid = normalizeJobId(tostring(info.job_id)) end
    if not pid and info.place_id then pid = tonumber(info.place_id) end
    if jid then
      showJoinInfo({ pet_name = info.pet_name, value = info.value, source = "List" }, jid, pid)
      teleport(jid, pid)
    end
  end)

  -- double-click row to join quickly
  local lastClick = 0
  row.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
      local now = tick()
      if now - lastClick < 0.25 then
        local jid, pid = nil, nil
        if info.join_script then jid, pid = parseJoinScript(info.join_script) end
        if not jid and info.job_id then jid = normalizeJobId(tostring(info.job_id)) end
        if not pid and info.place_id then pid = tonumber(info.place_id) end
        if jid then
          showJoinInfo({ pet_name = info.pet_name, value = info.value, source = "Double-Click" }, jid, pid)
          teleport(jid, pid)
        end
      end
      lastClick = now
    end
  end)

  return row
end

local function refreshList()
  -- servers snapshot
  local ok, res = pcall(function() return http_call({Url = API.."/servers", Method = "GET"}) end)
  if ok and res and res.StatusCode == 200 then
    local data = jsonDecode(res.Body or "") or {}
    local items = data.servers or {}
    if currentTab == "servers" then
      for _, child in ipairs(Scroller:GetChildren()) do
        if child:IsA("Frame") then child:Destroy() end
      end
      local count = 0
      for _, info in ipairs(items) do
        local nm = tostring(info.pet_name or "")
        if not (blacklistOn and inBlacklist(nm)) and (not string.find(string.lower(nm), "brainrot notify")) then
          local row = makeRow(info)
          row.Parent = Scroller
          count += 1
        end
      end
      Scroller.CanvasSize = UDim2.new(0, 0, 0, count * (40 + 8))
    end
  end
  -- multi pets bullets
  local ok2, res2 = pcall(function() return http_call({Url = API.."/multipets_detail", Method = "GET"}) end)
  if ok2 and res2 and res2.StatusCode == 200 and currentTab == "multi" then
    local d2 = jsonDecode(res2.Body or "") or {}
    if d2.items then
      for _, child in ipairs(mpList:GetChildren()) do if child:IsA("TextLabel") then child:Destroy() end end
      local shown = 0
      for _, it in ipairs(d2.items) do
        if shown >= 10 then break end
        local name = tostring(it.name or "Unknown")
        local v = tonumber(it.value or 0) or 0
        local vs = v > 0 and ("$"..string.format("%.1f", v/1e6).."M/s") or "--"
        local l = Instance.new("TextLabel")
        l.BackgroundTransparency = 1
        l.Size = UDim2.new(1, 0, 0, 18)
        l.Font = Enum.Font.Gotham
        l.TextSize = 14
        l.TextXAlignment = Enum.TextXAlignment.Left
        l.TextColor3 = Color3.fromRGB(190,190,190)
        l.Text = "•  "..name.."  –  "..vs
        l.Parent = mpList
        shown = shown + 1
      end
    end
  end
end

-- periodic UI refresh
local function showServersTab()
  currentTab = "servers"
  rowHeader.Visible = true
  Scroller.Visible = true
  mpBox.Visible = false
  refreshList()
end
local function showMultiTab()
  currentTab = "multi"
  rowHeader.Visible = false
  Scroller.Visible = false
  mpBox.Visible = true
  refreshList()
end
tabBtnServers.MouseButton1Click:Connect(showServersTab)
tabBtnMulti.MouseButton1Click:Connect(showMultiTab)

task.spawn(function()
  while true do
    task.wait(1.0)
    refreshList()
  end
end)

-- Command bridge polling: react to desktop log actions
task.spawn(function()
  while true do
    task.wait(0.25)
    local ok,res = pcall(function() return http_call({ Url = API.."/command", Method = "GET" }) end)
    if ok and res and res.StatusCode == 200 and res.Body then
      local d = jsonDecode(res.Body)
      if d and d.ok and not d.empty then
        local cmd = d.command or {}
        local act = string.lower(tostring(cmd.action or ""))
        if act == "join" and cmd.job_id then
          local jid = normalizeJobId(tostring(cmd.job_id))
          -- spam teleport a few times for reliability
          showJoinInfo({ source = "Desktop" }, jid, nil)
          task.spawn(function() for i=1,3 do teleport(jid) task.wait(0.15) end end)
        elseif act == "blacklist" and cmd.name then
          addBlacklist(tostring(cmd.name))
          rebuildBlacklist(); renderBlacklistChips()
          -- Optionally purge server queue to remove blacklisted
          pcall(function() return http_call({ Url = API.."/purge", Method = "POST", Headers = { ["Content-Type"] = "application/json" }, Body = _blJsonCache }) end)
        end
      end
    end
  end
end)

protect_gui(ScreenGui)

-- Start idle; user can click RUNNING then toggle Auto
setRunning(false)
setAuto(false)
showServersTab()

-- Loader removed: using native UI only; blacklist box replaces loader section
