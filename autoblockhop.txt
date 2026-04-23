task.wait(5)
-- ========== AUTO BLOCK HOP STANDALONE ==========
if not isfile("blockaccountlist.txt") then
    writefile("blockaccountlist.txt", "")
    print("📄 Đã tạo file blockaccountlist.txt")
end

local Players = game:GetService("Players")
local HttpService = game:GetService("HttpService")
local TeleportService = game:GetService("TeleportService")
local localPlayer = Players.LocalPlayer

-- ========== CONFIG ==========
local BLOCK_FILE        = "blockaccountlist.txt"
local HOP_DELAY         = 10
local MIN_PLAYERS_PREF  = 15
local MIN_PLAYERS_FALL  = 8
local MAX_PREF_ATTEMPTS = 3

-- ========== HARDCODED LIST ==========
local TARGET_NAMES = {
    "qtagawaawry68","392esnooty","ponzeruganze8","qdeseamudal","sidlowmmouse42",
    "coolsnlesch","7534wmcewen","lxagarpar21","yoppcpuz29","2669zevin",
    "mozleyc443","bearyeurzua","tangivfero","tomkov94299","herveyj3255",
    "ubougeelong51","athanyskokan","dinkhajfiori9","ujustynwee","kambojubono",
    "okralinjiao","9897opikle","veringotaut12","abaiq914","eloahkalic",
    "bthoeath","binhy942","cutrerqsereno","ysebdenkiss9","01655ldindle",
    "88332ispot","dilkerctge58","bitahzstreed43","rsannahseer94","167vanniek",
    "sipplfhevey52","refazojrutman","goudpcatino7","zercklwhit","gehannkotara",
    "parramnnunery4","baniaidora1","cvoorentew","413gshiqin","oryallkschmit",
    "bodlenlagory3","98556qdownby","meysonmurino","76707ddashee","kamatomeach",
    "jmuzarivip","hebeinjdizon84","lzhudisea","genaropoieda25","ripostfozy",
    "nfloriwean","tariqxloeven","coyaw62638","npastelwye72","maelystgagner13",
    "bpaolicoin20","tweetk1536","munciewabbasi","63774peither","arnasq142",
    "nemicann","412rcapot","8959flovee","8565gsajous","rackelj196",
    "11160hpudent","47327jfinch","nothernixing56","zelantseija","940yfonz",
    "28194pbacule","horleeradio10","bucketlekha","chaeresipo","wkitbagvaut",
    "lokkemarloe20","265tpilfer","garentzcap","kinnely3467","0880ydjinns",
    "uncribsubj","22162emugget","overszhartl92","nappoelykens37","stukkalan",
    "choirsbeltie","olannagdru","zipperbfefer53","sieglvfeener7","hasiky1297",
    "73634jzeid","mastejjmcniel6","mullgaily","945xcotoin","kochaylacau1",
    "viragawayt","14475nvorn","fwhutelava15","rarygschaap","bajetjbalmir4",
    "ettaleisner78","knotekaprewer9","54070nbilars","jtenuisran82","wadoodpjaffe2",
    "macrisyblose","abbsdhatman73","bluthd833","ktealrye","sjacenbeef",
    "wislervcapetl29","ohourpack87","wenseld9119","chidiantei","zmoonenkale11",
    "hoholu73432","huet75267","seyalc58753","elven2948","lighty38239",
    "clechyfrill","texp612","eliottu47314","antheszbeltz9","rachalpduong9",
    "earmanrspeedy25","vribosewatt","bearq46123","ndryadsnil88","dcoderdit",
    "viljamoleak15","gitaufboaz4","qjobstbane","3190hruffs","medillxflye",
}

-- ========== LOAD FILE ==========
local TXT_NAMES = {}  -- chỉ acc từ file txt

local function loadBlockList()
    print("✅ Hardcoded: "..#TARGET_NAMES.." accounts")
    local ok, content = pcall(readfile, BLOCK_FILE)
    if ok and content and content ~= "" then
        local added = 0
        for line in content:gmatch("[^\r\n]+") do
            local trimmed = line:gsub("^%s+", ""):gsub("%s+$", "")
            if trimmed ~= "" then
                local found = false
                for _, name in ipairs(TARGET_NAMES) do
                    if name:lower() == trimmed:lower() then found = true break end
                end
                if not found then
                    table.insert(TARGET_NAMES, trimmed)
                    table.insert(TXT_NAMES, trimmed:lower())  -- lưu riêng
                    added += 1
                end
            end
        end
        if added > 0 then
            print("📄 Thêm "..added.." acc từ file txt")
        end
    end
    print("📋 Tổng: "..#TARGET_NAMES.." blocked accounts")
end

-- ========== SNAPSHOT ==========
local existingPlayers = {}
local snapshotDone = false

local function snapshotCurrentPlayers()
    for _, player in ipairs(Players:GetPlayers()) do
        existingPlayers[player.Name:lower()] = true
    end
    local entities = workspace:FindFirstChild("Entities")
    if entities then
        for _, entity in ipairs(entities:GetChildren()) do
            existingPlayers[entity.Name:lower()] = true
        end
    end
    print("📸 Snapshot "..#Players:GetPlayers().." players – sẽ bỏ qua")
end

local function isBlockedAccount(name)
    if name:lower() == localPlayer.Name:lower() then return false end
    for _, blocked in ipairs(TARGET_NAMES) do
        if name:lower() == blocked:lower() then return true end
    end
    return false
end

-- Acc từ txt thì không bị snapshot chặn
local function isTxtAccount(name)
    for _, n in ipairs(TXT_NAMES) do
        if name:lower() == n then return true end
    end
    return false
end

-- ========== CHECK CÒN TRONG SERVER ==========
local function isStillInServer(playerName)
    local entities = workspace:FindFirstChild("Entities")
    if entities and entities:FindFirstChild(playerName) then return true end
    for _, player in ipairs(Players:GetPlayers()) do
        if player.Name:lower() == playerName:lower() then return true end
    end
    return false
end

-- ========== HOP ==========
local isHopping = false

local function hopToServer(forceFallback)
    if isHopping then return end
    isHopping = true
    print(forceFallback and "🔄 Tìm server ít người (fallback)..." or "🔄 Tìm server nhiều người...")

    task.spawn(function()
        local BASE_URL = "https://games.roblox.com/v1/games/"..tostring(game.PlaceId).."/servers/Public?sortOrder=Desc&limit=100"
        local hopStart = tick()

        while true do
            local ok, result = pcall(function()
                local raw = game:HttpGet(BASE_URL)
                local data = HttpService:JSONDecode(raw)
                if not data or not data.data then return false end

                local preferred, fallback = {}, {}
                for _, s in ipairs(data.data) do
                    if s.id ~= game.JobId and s.playing < s.maxPlayers then
                        if s.playing >= MIN_PLAYERS_PREF then
                            table.insert(preferred, s)
                        elseif s.playing >= MIN_PLAYERS_FALL then
                            table.insert(fallback, s)
                        end
                    end
                end

                local pool = forceFallback and (#fallback > 0 and fallback or nil)
                    or (#preferred > 0 and preferred or nil)

                if not pool or #pool == 0 then
                    print("⚠️ Không tìm thấy server, thử lại...")
                    return false
                end

                local server = pool[math.random(1, #pool)]
                print("✅ Server found! Players: "..server.playing.."/"..server.maxPlayers)
                TeleportService:TeleportToPlaceInstance(game.PlaceId, server.id)
                return true
            end)

            if ok and result then break end
            if tick() - hopStart > 30 then warn("❌ Timeout 30s") break end
            task.wait(2)
        end

        isHopping = false
    end)
end

-- ========== TRIGGER HOP ==========
local pendingHop = false

local function triggerHop(playerName)
    if pendingHop or isHopping then return end
    pendingHop = true

    task.spawn(function()
        local prefAttempts = 0

        while prefAttempts < MAX_PREF_ATTEMPTS do
            prefAttempts += 1
            print("⚠️ "..playerName.." – lần "..prefAttempts.."/"..MAX_PREF_ATTEMPTS.." – chờ "..HOP_DELAY.."s...")
            task.wait(HOP_DELAY)

            if not isStillInServer(playerName) then
                print("✅ "..playerName.." đã out")
                pendingHop = false
                return
            end

            print("🔄 Hopping server nhiều người (lần "..prefAttempts..")...")
            hopToServer(false)

            local waitStart = tick()
            while isHopping and tick() - waitStart < 35 do
                task.wait(1)
            end
        end

        print("⚠️ Hết "..MAX_PREF_ATTEMPTS.." lần – fallback server ít người...")
        hopToServer(true)
        pendingHop = false
    end)
end

-- ========== PLAYER ADDED ==========
Players.PlayerAdded:Connect(function(player)
    if existingPlayers[player.Name:lower()] then return end
    if isBlockedAccount(player.Name) then
        print("🚨 "..player.Name.." vừa JOIN!")
        triggerHop(player.Name)
    end
end)

-- ========== CHECK ENTITIES MỖI 3S ==========
task.spawn(function()
    while true do
        task.wait(3)
        if not snapshotDone then continue end
        if pendingHop or isHopping then continue end
        local entities = workspace:FindFirstChild("Entities")
        if not entities then continue end
        for _, entity in ipairs(entities:GetChildren()) do
            if not existingPlayers[entity.Name:lower()] and isBlockedAccount(entity.Name) then
                print("🚨 Phát hiện qua Entities: "..entity.Name)
                triggerHop(entity.Name)
                break
            end
        end
    end
end)

-- ========== MAIN ==========
loadBlockList()

task.spawn(function()
    task.wait(5)
    snapshotCurrentPlayers()
    snapshotDone = true 
    
    local foundExisting = {}
    for _, name in ipairs(TARGET_NAMES) do
        if existingPlayers[name:lower()] then
            table.insert(foundExisting, name)
        end
    end
    local skipped = {}
    local triggerNow = {}
    for _, name in ipairs(foundExisting) do
        if isTxtAccount(name) then
            table.insert(triggerNow, name)  -- txt acc → hop luôn
        else
            table.insert(skipped, name)     -- hardcoded → bỏ qua
        end
    end
    if #skipped > 0 then
        print("⚠️ Hardcoded có sẵn (bỏ qua): "..table.concat(skipped, ", "))
    end
    if #triggerNow > 0 then
        print("🚨 Txt acc có sẵn → trigger hop: "..table.concat(triggerNow, ", "))
        triggerHop(triggerNow[1])
    end
    if #skipped == 0 and #triggerNow == 0 then
        print("✅ Server hiện tại sạch")
    end

    print("━━━━━━━━━━━━━━━━━━━━━━━━━")
    print("✅ AutoBlockHop đang chạy, đợi tí thằng lồn")
    print("⏱️ Đợi xí: "..HOP_DELAY.."s | Attempts: "..MAX_PREF_ATTEMPTS)
    print("👥 Số người >"..MIN_PLAYERS_PREF.." | Fallback "..MIN_PLAYERS_FALL.."-"..MIN_PLAYERS_PREF)
    print("━━━━━━━━━━━━━━━━━━━━━━━━━")
end)