-- Ping Favorites v3.4 - Custom UI Window (touch friendly) By Aartvark Creative (Gavin Lake)
-- Changes:
--  - Info line (page/devices/source/pings) is explicitly ABOVE the grid and left-aligned
--  - Added "Conn" column showing last port used for that device (AUTO/Con1/Con2/Con3)
--  - Moved CLR one column to the right (after Conn)
--  - Feel free to use and distribute this 
--  - Backward-compatible save format: old entries without Conn default to "-"

local pluginName    = select(1, ...)
local componentName = select(2, ...)
local signalTable   = select(3, ...)
local myHandle      = select(4, ...)

-- -----------------------------
-- Persistent keys
-- -----------------------------
local VAR_KEY_LIST   = "PingFavorites_v3_list"
local VAR_KEY_PORT   = "PingFavorites_v3_port"     -- "AUTO" | "CON1" | "CON2" | "CON3"
local VAR_KEY_CON1IP = "PingFavorites_v3_con1_ip"
local VAR_KEY_CON2IP = "PingFavorites_v3_con2_ip"
local VAR_KEY_CON3IP = "PingFavorites_v3_con3_ip"
local VAR_KEY_PAGE   = "PingFavorites_v3_page"

local DEFAULT_PING_COUNT = 5

-- UI sizing
local PAGE_SIZE   = 10
local ROW_HEIGHT  = "52"
local FONT_NAMEIP = "Regular18"
local FONT_HDR    = "Medium20"
local FONT_BTN    = "Medium20"
local FONT_INFO   = "Regular18"

-- -----------------------------
-- Utils
-- -----------------------------
local function trim(s)
  return (tostring(s or ""):gsub("^%s+", ""):gsub("%s+$", ""))
end

local function is_ipv4(ip)
  local a,b,c,d = tostring(ip or ""):match("^(%d+)%.(%d+)%.(%d+)%.(%d+)$")
  a,b,c,d = tonumber(a),tonumber(b),tonumber(c),tonumber(d)
  if not (a and b and c and d) then return false end
  local function ok(x) return x >= 0 and x <= 255 end
  return ok(a) and ok(b) and ok(c) and ok(d)
end

local function now_epoch()
  if os and os.time then return os.time() end
  return 0
end

-- -----------------------------
-- UserVars
-- -----------------------------
local function load_var(key)
  local vs = UserVars()
  return GetVar(vs, key)
end

local function save_var(key, value)
  local vs = UserVars()
  SetVar(vs, key, value)
end

-- List encoding (v3.4):
-- ip \t name \t last_ok(1/0) \t last_avg_ms \t last_ts \t last_port
-- Backward compatible with old 5-field lines (no last_port).
local function encode_list(items)
  local lines = {}
  for _, e in ipairs(items) do
    local ip   = trim(e.ip)
    local name = tostring(e.name or ""):gsub("[\t\r\n]", " ")
    if ip ~= "" then
      local ok   = (e.last_ok and "1" or "0")
      local ms   = tostring(e.last_ms or "")
      local ts   = tostring(e.last_ts or 0)
      local port = tostring(e.last_port or ""):gsub("[\t\r\n]", " ")
      table.insert(lines, ip .. "\t" .. name .. "\t" .. ok .. "\t" .. ms .. "\t" .. ts .. "\t" .. port)
    end
  end
  return table.concat(lines, "\n")
end

local function decode_list(raw)
  local items = {}
  raw = tostring(raw or "")
  if raw == "" then return items end

  for line in raw:gmatch("[^\n]+") do
    -- Try v3.4 (6 fields)
    local ip, name, ok, ms, ts, port = line:match("^(.-)\t(.-)\t(.-)\t(.-)\t(.-)\t(.-)$")
    if not ip then
      -- Try old format (5 fields)
      ip, name, ok, ms, ts = line:match("^(.-)\t(.-)\t(.-)\t(.-)\t(.-)$")
      port = ""
    end

    ip = trim(ip)
    if ip ~= "" then
      table.insert(items, {
        ip = ip,
        name = trim(name),
        last_ok = (trim(ok) == "1"),
        last_ms = trim(ms),
        last_ts = tonumber(ts) or 0,
        last_port = trim(port),
      })
    end
  end

  return items
end

local function load_favs()
  return decode_list(load_var(VAR_KEY_LIST))
end

local function save_favs(items)
  save_var(VAR_KEY_LIST, encode_list(items))
end

local function sort_favs(items)
  table.sort(items, function(a, b)
    local an = (a.name ~= "" and a.name or a.ip):lower()
    local bn = (b.name ~= "" and b.name or b.ip):lower()
    return an < bn
  end)
end

local function find_by_ip(items, ip)
  for i, e in ipairs(items) do
    if e.ip == ip then return i end
  end
  return nil
end

-- -----------------------------
-- Prompts
-- -----------------------------
local function mb(title, message, commands)
  return MessageBox({
    title = title,
    message = message,
    commands = commands or { { value = 1, name = "OK" } }
  })
end

local function prompt(title, default)
  local r = TextInput(title, default or "")
  if r == nil then return nil end
  r = trim(r)
  if r == "" then return nil end
  return r
end

-- -----------------------------
-- Ping runner
-- -----------------------------
local function try_popen(cmd)
  if type(io) ~= "table" or type(io.popen) ~= "function" then return nil end
  local ok, pipe = pcall(io.popen, cmd .. " 2>&1")
  if not ok or not pipe then return nil end
  local out = pipe:read("*a")
  pipe:close()
  return out
end

local function read_file(path)
  local f = io.open(path, "r")
  if not f then return nil end
  local data = f:read("*a")
  f:close()
  return data
end

local function run_cmd_capture(cmd)
  local out = try_popen(cmd)
  if out and out ~= "" then return out end

  local tmp = os.tmpname()
  local outfile = tmp .. "_ma3_ping.txt"
  local redir_cmd = string.format('%s > "%s" 2>&1', cmd, outfile)

  if type(os) == "table" and type(os.execute) == "function" then
    pcall(os.execute, redir_cmd)
  end

  local file_out = read_file(outfile)
  pcall(os.remove, outfile)
  return file_out
end

-- MA console ping output (BSD/mac style)
local function parse_ping(out)
  out = tostring(out or "")

  local recv = out:match("(%d+)%s+packets%s+received")
  if recv then
    recv = tonumber(recv)
    if recv and recv > 0 then
      local avg = out:match("round%-trip.-=.-/([%d%.]+)/")
      return true, avg
    else
      return false, nil
    end
  end

  if out:match("[Bb]ytes%s+from") or out:match("[Rr]eply%s+from") then
    local avg = out:match("round%-trip.-=.-/([%d%.]+)/")
    return true, avg
  end

  return false, nil
end

local function get_selected_port()
  local p = trim(load_var(VAR_KEY_PORT))
  if p == "" then p = "AUTO" end
  return p
end

local function set_selected_port(p)
  save_var(VAR_KEY_PORT, p)
end

local function get_source_ip_for_port(port)
  port = tostring(port or "AUTO")
  if port == "CON1" then return trim(load_var(VAR_KEY_CON1IP)) end
  if port == "CON2" then return trim(load_var(VAR_KEY_CON2IP)) end
  if port == "CON3" then return trim(load_var(VAR_KEY_CON3IP)) end
  return ""
end

local function build_ping_commands(ip, count, port)
  count = tonumber(count) or DEFAULT_PING_COUNT
  if count < 1 then count = 1 end
  if count > 20 then count = 20 end

  local src = get_source_ip_for_port(port)
  local cmds = {}

  if port ~= "AUTO" and is_ipv4(src) then
    table.insert(cmds, string.format('ping -I %s -c %d %s', src, count, ip))
    table.insert(cmds, string.format('ping -I %s -n %d %s', src, count, ip))
  end

  table.insert(cmds, string.format('ping -c %d %s', count, ip))
  table.insert(cmds, string.format('ping -n %d %s', count, ip))

  return cmds, src
end

local function ping_ip(ip)
  local port = get_selected_port()
  local candidates, src = build_ping_commands(ip, DEFAULT_PING_COUNT, port)

  for _, cmd in ipairs(candidates) do
    local out = run_cmd_capture(cmd)
    if out and out ~= "" then
      local ok, avg = parse_ping(out)
      return cmd, out, ok, avg, port, src
    end
  end

  return "(no command ran)", "Unable to run ping (or capture output).", false, nil, port, src
end

-- -----------------------------
-- Data actions
-- -----------------------------
local function add_device_flow()
  local ip = prompt("Add device IP", "192.168.0.10")
  if not ip then return end
  if not is_ipv4(ip) then
    mb("Ping Favorites", "Invalid IPv4 address:\n" .. ip)
    return
  end

  local name = prompt("Device name (optional)", "FOH Switch") or ""

  local items = load_favs()
  local idx = find_by_ip(items, ip)
  if idx then
    if name ~= "" then items[idx].name = name end
  else
    table.insert(items, { ip = ip, name = name, last_ok = false, last_ms = "", last_ts = 0, last_port = "" })
  end

  sort_favs(items)
  save_favs(items)
end

local function rename_device(ip)
  local items = load_favs()
  local idx = find_by_ip(items, ip)
  if not idx then return end

  local current = items[idx].name
  local newname = prompt("Rename " .. ip, (current ~= "" and current or "Device name"))
  if not newname then return end

  items[idx].name = newname
  sort_favs(items)
  save_favs(items)
end

local function delete_device_confirm(ip)
  local items = load_favs()
  local idx = find_by_ip(items, ip)
  if not idx then return end

  local e = items[idx]
  local label = (e.name ~= "" and (e.name .. " (" .. e.ip .. ")") or e.ip)

  local r = mb(
    "Remove device?",
    "Remove:\n" .. label,
    { { value = 1, name = "Cancel" }, { value = 2, name = "Remove" } }
  )
  if r and r.result == 2 then
    table.remove(items, idx)
    save_favs(items)
  end
end

local function clear_all_confirm()
  local items = load_favs()
  if #items == 0 then
    mb("Clear All", "Favorites list is already empty.")
    return
  end

  local r = mb(
    "Clear ALL favorites?",
    "This will remove ALL stored devices (" .. tostring(#items) .. ").",
    { { value = 1, name = "Cancel" }, { value = 2, name = "Clear All" } }
  )
  if r and r.result == 2 then
    save_var(VAR_KEY_LIST, "")
    save_var(VAR_KEY_PAGE, "1")
  end
end

local function edit_source_ips_flow()
  local con1 = trim(load_var(VAR_KEY_CON1IP))
  local con2 = trim(load_var(VAR_KEY_CON2IP))
  local con3 = trim(load_var(VAR_KEY_CON3IP))

  local r = MessageBox({
    title = "Edit Source IPs",
    message =
      "Set these to the IPs assigned in Menu > Network > Interfaces.\n\n" ..
      "Con1: " .. (con1 ~= "" and con1 or "(not set)") .. "\n" ..
      "Con2: " .. (con2 ~= "" and con2 or "(not set)") .. "\n" ..
      "Con3: " .. (con3 ~= "" and con3 or "(not set)") .. "\n",
    commands = {
      { value = 1, name = "Back" },
      { value = 2, name = "Edit Con1" },
      { value = 3, name = "Edit Con2" },
      { value = 4, name = "Edit Con3" },
    }
  })
  if not r then return end

  if r.result == 2 then
    local ip = prompt("Con1 source IP", con1 ~= "" and con1 or "192.168.0.2")
    if ip then
      if is_ipv4(ip) then save_var(VAR_KEY_CON1IP, ip) else mb("Edit Source IPs", "Invalid IPv4:\n" .. ip) end
    end
  elseif r.result == 3 then
    local ip = prompt("Con2 source IP", con2 ~= "" and con2 or "10.0.0.2")
    if ip then
      if is_ipv4(ip) then save_var(VAR_KEY_CON2IP, ip) else mb("Edit Source IPs", "Invalid IPv4:\n" .. ip) end
    end
  elseif r.result == 4 then
    local ip = prompt("Con3 source IP", con3 ~= "" and con3 or "172.16.0.2")
    if ip then
      if is_ipv4(ip) then save_var(VAR_KEY_CON3IP, ip) else mb("Edit Source IPs", "Invalid IPv4:\n" .. ip) end
    end
  end
end

-- -----------------------------
-- Page helpers
-- -----------------------------
local function get_page()
  local p = tonumber(load_var(VAR_KEY_PAGE) or "1") or 1
  if p < 1 then p = 1 end
  return p
end

local function set_page(p)
  p = tonumber(p) or 1
  if p < 1 then p = 1 end
  save_var(VAR_KEY_PAGE, tostring(p))
end

-- -----------------------------
-- Render window
-- -----------------------------
local function render()
  local displayIndex = Obj.Index(GetFocusDisplay())
  if displayIndex > 5 then displayIndex = 1 end
  local display = GetDisplayByIndex(displayIndex)
  local screenOverlay = display.ScreenOverlay

  local cTransparent = Root().ColorTheme.ColorGroups.Global.Transparent
  local cBtnBg       = Root().ColorTheme.ColorGroups.Button.Background
  local cOk          = Root().ColorTheme.ColorGroups.Global.PartlySelectedPreset
  local cFail        = Root().ColorTheme.ColorGroups.Global.PartlySelected
  local cSelect      = Root().ColorTheme.ColorGroups.Global.Selected

  local items = load_favs()
  sort_favs(items)

  local page = get_page()
  local total = #items
  local total_pages = math.max(1, math.ceil(total / PAGE_SIZE))
  if page > total_pages then page = total_pages end
  set_page(page)

  local start_i = (page - 1) * PAGE_SIZE + 1
  local end_i = math.min(total, start_i + PAGE_SIZE - 1)

  screenOverlay:ClearUIChildren()

  local baseInput = screenOverlay:Append("BaseInput")
  baseInput.Name = "PingFavoritesWindow"
  baseInput.W, baseInput.H = "1120", "720"
  baseInput.AutoClose = "No"
  baseInput.CloseOnEscape = "Yes"
  baseInput.Columns = 1
  baseInput.Rows = 2
  baseInput[1][1].SizePolicy = "Fixed"
  baseInput[1][1].Size = "75"
  baseInput[1][2].SizePolicy = "Stretch"

  -- Title bar with port selector
  local titleBar = baseInput:Append("TitleBar")
  titleBar.Columns = 6
  titleBar.Rows = 1
  titleBar.Anchors = "0,0"
  titleBar[2][2].SizePolicy = "Stretch"
  titleBar[2][3].SizePolicy = "Fixed"; titleBar[2][3].Size = "120"
  titleBar[2][4].SizePolicy = "Fixed"; titleBar[2][4].Size = "120"
  titleBar[2][5].SizePolicy = "Fixed"; titleBar[2][5].Size = "120"
  titleBar[2][6].SizePolicy = "Fixed"; titleBar[2][6].Size = "120"

  local titleBtn = titleBar:Append("TitleButton")
  titleBtn.Text = "Ping Favorites"
  titleBtn.Icon = "network"

  local function port_btn(txt, col, value)
    local b = titleBar:Append("Button")
    b.Text = txt
    b.Font = FONT_BTN
    b.Textshadow = 1
    b.TextalignmentH = "Centre"
    b.PluginComponent = myHandle
    b.Clicked = "SetPort_" .. value
    b.BackColor = (get_selected_port() == value) and cSelect or cBtnBg
    b.Anchors = { left = col-1, right = col-1, top = 0, bottom = 0 }
    return b
  end

  port_btn("AUTO", 3, "AUTO")
  port_btn("Con1", 4, "CON1")
  port_btn("Con2", 5, "CON2")
  port_btn("Con3", 6, "CON3")

  -- Main frame: Info row ABOVE grid
  local frame = baseInput:Append("DialogFrame")
  frame.H = "100%"
  frame.W = "100%"
  frame.Columns = 1
  frame.Rows = 4
  frame.Anchors = { left = 0, right = 0, top = 1, bottom = 1 }

  frame[1][1].SizePolicy = "Fixed";  frame[1][1].Size = "44"  -- info row
  frame[1][2].SizePolicy = "Fixed";  frame[1][2].Size = "36"  -- header row (optional)
  frame[1][3].SizePolicy = "Stretch"                           -- grid
  frame[1][4].SizePolicy = "Fixed";  frame[1][4].Size = "95"  -- bottom buttons

  -- Info line (explicit anchors, left aligned)
  local info = frame:Append("UIObject")
  info.Anchors = { left = 0, right = 0, top = 0, bottom = 0 }
  info.Padding = { left = 18, right = 18, top = 8, bottom = 8 }
  info.Font = FONT_INFO
  info.TextalignmentH = "Left"
  info.TextalignmentV = "Center"
  info.HasHover = "No"
  info.BackColor = cTransparent
  info.Text = string.format(
    "Page %d/%d   •   Devices: %d   •   Source: %s   •   Pings: %d",
    page, total_pages, total, get_selected_port(), DEFAULT_PING_COUNT
  )

  -- Column header line (so the grid header doesn't get visually buried)
  local hdr = frame:Append("UIObject")
  hdr.Anchors = { left = 0, right = 0, top = 1, bottom = 1 }
  hdr.Padding = { left = 18, right = 18, top = 6, bottom = 6 }
  hdr.Font = FONT_HDR
  hdr.TextalignmentH = "Left"
  hdr.TextalignmentV = "Center"
  hdr.HasHover = "No"
  hdr.BackColor = cTransparent
  hdr.Text = "St      Name                                   IP                      Avg      Ping    Conn   CLR   …"

  -- Grid: now includes Conn column, CLR shifted right
  local grid = frame:Append("UILayoutGrid")
  grid.Columns = 14
  grid.Rows = PAGE_SIZE
  grid.Anchors = { left = 0, right = 0, top = 2, bottom = 2 }
  grid.Margin = { left = 10, right = 10, top = 0, bottom = 0 }

  for r = 0, PAGE_SIZE do
    grid["Row" .. tostring(r)] = ROW_HEIGHT
  end

  local rowToIp = {}
  local row = 0
  for idx = start_i, end_i do
    local e = items[idx]
    local ip = e.ip
    rowToIp[row + 1] = ip

    -- Status box
    local st = grid:Append("Button")
    st.Anchors = { left = 0, right = 0, top = row, bottom = row }
    st.Text = ""
    st.HasHover = "No"
    if e.last_ts and e.last_ts > 0 then
      st.BackColor = e.last_ok and cOk or cFail
    else
      st.BackColor = cBtnBg
    end

    -- Name
    local name = grid:Append("UIObject")
    name.Anchors = { left = 1, right = 6, top = row, bottom = row }
    name.Padding = "10,8"
    name.Font = FONT_NAMEIP
    name.TextalignmentH = "Left"
    name.TextalignmentV = "Center"
    name.HasHover = "No"
    name.BackColor = cTransparent
    name.Text = (e.name ~= "" and e.name or "(unnamed)")

    -- IP
    local ipObj = grid:Append("UIObject")
    ipObj.Anchors = { left = 7, right = 8, top = row, bottom = row }
    ipObj.Padding = "10,8"
    ipObj.Font = FONT_NAMEIP
    ipObj.TextalignmentH = "Left"
    ipObj.TextalignmentV = "Center"
    ipObj.HasHover = "No"
    ipObj.BackColor = cTransparent
    ipObj.Text = ip

    -- Avg
    local avgObj = grid:Append("UIObject")
    avgObj.Anchors = { left = 9, right = 9, top = row, bottom = row }
    avgObj.Padding = "10,8"
    avgObj.Font = FONT_NAMEIP
    avgObj.TextalignmentH = "Left"
    avgObj.TextalignmentV = "Center"
    avgObj.HasHover = "No"
    avgObj.BackColor = cTransparent
    avgObj.Text = (e.last_ms ~= "" and (e.last_ms .. "ms") or "-")

    -- Ping button
    local pingBtn = grid:Append("Button")
    pingBtn.Anchors = { left = 10, right = 10, top = row, bottom = row }
    pingBtn.Text = "PING"
    pingBtn.Font = FONT_BTN
    pingBtn.Textshadow = 1
    pingBtn.TextalignmentH = "Centre"
    pingBtn.PluginComponent = myHandle
    pingBtn.Clicked = "PingRow_" .. tostring(row + 1)

    -- Conn column (last port used)
    local connObj = grid:Append("UIObject")
    connObj.Anchors = { left = 11, right = 11, top = row, bottom = row }
    connObj.Padding = "10,8"
    connObj.Font = FONT_NAMEIP
    connObj.TextalignmentH = "Centre"
    connObj.TextalignmentV = "Center"
    connObj.HasHover = "No"
    connObj.BackColor = cTransparent
    local lp = trim(e.last_port)
    if lp == "" then lp = "-" end
    if lp == "CON1" then lp = "Con1" end
    if lp == "CON2" then lp = "Con2" end
    if lp == "CON3" then lp = "Con3" end
    connObj.Text = lp

    -- CLR button (shifted right)
    local clrBtn = grid:Append("Button")
    clrBtn.Anchors = { left = 12, right = 12, top = row, bottom = row }
    clrBtn.Text = "CLR"
    clrBtn.Font = FONT_BTN
    clrBtn.Textshadow = 1
    clrBtn.TextalignmentH = "Centre"
    clrBtn.PluginComponent = myHandle
    clrBtn.Clicked = "ClearRow_" .. tostring(row + 1)

    -- More
    local moreBtn = grid:Append("Button")
    moreBtn.Anchors = { left = 13, right = 13, top = row, bottom = row }
    moreBtn.Text = "…"
    moreBtn.Font = FONT_BTN
    moreBtn.Textshadow = 1
    moreBtn.TextalignmentH = "Centre"
    moreBtn.PluginComponent = myHandle
    moreBtn.Clicked = "MoreRow_" .. tostring(row + 1)

    row = row + 1
  end

  -- Fill remaining rows
  while row < PAGE_SIZE do
    local filler = grid:Append("UIObject")
    filler.Anchors = { left = 0, right = 13, top = row, bottom = row }
    filler.Text = ""
    filler.HasHover = "No"
    filler.BackColor = cTransparent
    row = row + 1
  end

  -- Bottom controls
  local bottom = frame:Append("UILayoutGrid")
  bottom.Columns = 7
  bottom.Rows = 1
  bottom.Anchors = { left = 0, right = 0, top = 3, bottom = 3 }
  bottom.Margin = { left = 10, right = 10, top = 5, bottom = 10 }

  local function bottom_btn(text, col, handler)
    local b = bottom:Append("Button")
    b.Anchors = { left = col, right = col, top = 0, bottom = 0 }
    b.Text = text
    b.Font = FONT_BTN
    b.Textshadow = 1
    b.TextalignmentH = "Centre"
    b.PluginComponent = myHandle
    b.Clicked = handler
    return b
  end

  bottom_btn("◀ Prev",     0, "PrevPage")
  bottom_btn("Next ▶",     1, "NextPage")
  bottom_btn("Add",        2, "AddDevice")
  bottom_btn("Edit IPs",   3, "EditSourceIPs")
  bottom_btn("Clear All",  4, "ClearAll")
  bottom_btn("Refresh",    5, "RefreshUI")
  bottom_btn("Close",      6, "CloseUI")

  -- -----------------------------
  -- Signal handlers
  -- -----------------------------
  signalTable.CloseUI = function() screenOverlay:ClearUIChildren() end
  signalTable.RefreshUI = function() render() end

  signalTable.PrevPage = function()
    local p = get_page()
    if p > 1 then set_page(p - 1) end
    render()
  end

  signalTable.NextPage = function()
    local p = get_page()
    if p < total_pages then set_page(p + 1) end
    render()
  end

  signalTable.AddDevice = function()
    add_device_flow()
    render()
  end

  signalTable.EditSourceIPs = function()
    edit_source_ips_flow()
    render()
  end

  signalTable.ClearAll = function()
    clear_all_confirm()
    render()
  end

  -- Port selector handlers
  signalTable.SetPort_AUTO = function() set_selected_port("AUTO"); render() end
  signalTable.SetPort_CON1 = function() set_selected_port("CON1"); render() end
  signalTable.SetPort_CON2 = function() set_selected_port("CON2"); render() end
  signalTable.SetPort_CON3 = function() set_selected_port("CON3"); render() end

  -- Row handlers
  for r = 1, PAGE_SIZE do
    local ip = rowToIp[r]
    if ip then
      signalTable["PingRow_" .. tostring(r)] = function()
        local cmd, out, ok, avg, port = ping_ip(ip)

        local items2 = load_favs()
        local idx2 = find_by_ip(items2, ip)
        if idx2 then
          items2[idx2].last_ok = ok
          items2[idx2].last_ms = avg or ""
          items2[idx2].last_ts = now_epoch()
          items2[idx2].last_port = port or ""
          save_favs(items2)
        end

        local title = ok and ("🟩 " .. ip) or ("🟥 " .. ip)
        mb(title, "Avg: " .. (avg and (avg .. "ms") or "-") .. "\n\n" .. cmd .. "\n\n" .. out)
        render()
      end

      signalTable["ClearRow_" .. tostring(r)] = function()
        delete_device_confirm(ip)
        render()
      end

      signalTable["MoreRow_" .. tostring(r)] = function()
        local res = MessageBox({
          title = "Device",
          message = ip,
          commands = {
            { value = 1, name = "Back" },
            { value = 2, name = "Rename" },
            { value = 3, name = "Remove" },
          }
        })
        if res and res.result == 2 then rename_device(ip) end
        if res and res.result == 3 then delete_device_confirm(ip) end
        render()
      end
    else
      signalTable["PingRow_" .. tostring(r)] = nil
      signalTable["ClearRow_" .. tostring(r)] = nil
      signalTable["MoreRow_" .. tostring(r)] = nil
    end
  end
end

-- Entry point
return function()
  render()
end
