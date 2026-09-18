local HttpService = game:GetService("HttpService")
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Workspace = game:GetService("Workspace")
local StarterGui = game:GetService("StarterGui")

local TRACKER_VERSION = "1.0.15"
local API_URL = "https://steal-an-egg-trackstats.vercel.app/api/update"
local API_KEY = "BatmanSAE_9xK72pQ2026"
local SEND_INTERVAL = 15
local SEND_DATA = true
local DEBUG_ICONS = false
local DEBUG_ICON_LIMIT = 80
local ICON_SYNC = true
local ICON_SIZE = 64
local ICON_SYNC_DELAY = 0.20
local ICON_UPLOAD_URL = API_URL:gsub("/api/update$", "/api/icon-upload")

repeat task.wait() until game:IsLoaded()
local Player = Players.LocalPlayer or Players.PlayerAdded:Wait()

local Request =
    request
    or http_request
    or (http and http.request)
    or (syn and syn.request)
    or (fluxus and fluxus.request)

if not Request then
    warn("SEASHOP: executor does not support request/http_request")
    return
end

local Env = getgenv and getgenv() or _G
local Config = type(Env.SEASHOP_SAE_CONFIG) == "table" and Env.SEASHOP_SAE_CONFIG or {}
Env.SEASHOP_SAE_CONFIG = Config

API_URL = tostring(Config.API_URL or API_URL)
API_KEY = tostring(Config.API_KEY or API_KEY)
SEND_INTERVAL = math.clamp(tonumber(Config.SEND_INTERVAL) or SEND_INTERVAL, 5, 3600)
SEND_DATA = Config.SEND_DATA ~= false

Config.API_URL = API_URL
Config.API_KEY = API_KEY
Config.SEND_INTERVAL = SEND_INTERVAL
Config.SEND_DATA = SEND_DATA
ICON_UPLOAD_URL = API_URL:gsub("/api/update$", "/api/icon-upload")

if Env.SEASHOP_SAE_TRACKER_STOP then
    pcall(Env.SEASHOP_SAE_TRACKER_STOP)
end

local Running = true
local Session = HttpService:GenerateGUID(false)

Env.SEASHOP_SAE_TRACKER_STOP = function()
    Running = false
end

local Save
local Assets = {}
local AssetItems
local EggState
local Mutations
local Rarities = {}

pcall(function()
    Save = require(ReplicatedStorage.Shared.Save)
end)

pcall(function()
    Assets = require(ReplicatedStorage.Data.Assets).Directory
end)

pcall(function()
    AssetItems = require(ReplicatedStorage.Shared.Util.AssetItems)
end)

pcall(function()
    EggState = require(ReplicatedStorage.Client.EggState)
end)

pcall(function()
    Mutations = require(ReplicatedStorage.Shared.Modules.Mutations)
end)

pcall(function()
    Rarities = require(ReplicatedStorage.Data.Rarity).Rarities
end)

local RarityById = {}

for _, Rarity in pairs(type(Rarities) == "table" and Rarities or {}) do
    if type(Rarity) == "table" and Rarity._id then
        RarityById[tostring(Rarity._id)] = Rarity
    end
end

local function getAssetId(Value)
    if type(Value) == "number" then
        return math.floor(Value)
    end

    local Text = tostring(Value or "")
    return tonumber(
        Text:match("rbxassetid://(%d+)")
        or Text:match("rbxthumb://.-[?&]id=(%d+)")
        or Text:match("[?&]id=(%d+)")
        or Text:match("^(%d+)$")
    ) or 0
end

local function colorToHex(Color)
    if typeof(Color) ~= "Color3" then
        return ""
    end

    return string.format(
        "#%02X%02X%02X",
        math.floor(Color.R * 255 + 0.5),
        math.floor(Color.G * 255 + 0.5),
        math.floor(Color.B * 255 + 0.5)
    )
end

local function getRawIcon(Value)
    if Value == nil then
        return ""
    end

    local Success, Text = pcall(function()
        return tostring(Value)
    end)

    if not Success then
        return ""
    end

    Text = tostring(Text or "")
    if #Text > 500 then
        Text = Text:sub(1, 500)
    end

    return Text
end


local function base64Encode(Data)
    local Alphabet = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"
    local Out = {}
    local Length = #Data
    local Index = 1

    while Index <= Length do
        local A = string.byte(Data, Index) or 0
        local B = string.byte(Data, Index + 1) or 0
        local C = string.byte(Data, Index + 2) or 0
        local Value = A * 65536 + B * 256 + C

        local I1 = math.floor(Value / 262144) % 64 + 1
        local I2 = math.floor(Value / 4096) % 64 + 1
        local I3 = math.floor(Value / 64) % 64 + 1
        local I4 = Value % 64 + 1

        Out[#Out + 1] = Alphabet:sub(I1, I1)
        Out[#Out + 1] = Alphabet:sub(I2, I2)
        Out[#Out + 1] = Index + 1 <= Length and Alphabet:sub(I3, I3) or "="
        Out[#Out + 1] = Index + 2 <= Length and Alphabet:sub(I4, I4) or "="

        Index += 3
    end

    return table.concat(Out)
end

local CRC32Table
local function getCRC32Table()
    if CRC32Table then
        return CRC32Table
    end

    CRC32Table = {}

    for I = 0, 255 do
        local C = I
        for _ = 1, 8 do
            if bit32.band(C, 1) == 1 then
                C = bit32.bxor(0xEDB88320, bit32.rshift(C, 1))
            else
                C = bit32.rshift(C, 1)
            end
        end
        CRC32Table[I] = C
    end

    return CRC32Table
end

local function crc32(Data)
    local Table = getCRC32Table()
    local CRC = 0xFFFFFFFF

    for I = 1, #Data do
        CRC = bit32.bxor(
            bit32.rshift(CRC, 8),
            Table[bit32.band(bit32.bxor(CRC, string.byte(Data, I)), 255)]
        )
    end

    return bit32.bxor(CRC, 0xFFFFFFFF)
end

local function adler32(Data)
    local A = 1
    local B = 0
    local Index = 1

    while Index <= #Data do
        local Last = math.min(Index + 3000, #Data)
        for I = Index, Last do
            A += string.byte(Data, I)
            B += A
        end
        A %= 65521
        B %= 65521
        Index = Last + 1
    end

    return B * 65536 + A
end

local function u32be(Value)
    return string.char(
        bit32.band(bit32.rshift(Value, 24), 255),
        bit32.band(bit32.rshift(Value, 16), 255),
        bit32.band(bit32.rshift(Value, 8), 255),
        bit32.band(Value, 255)
    )
end

local function pngChunk(Name, Data)
    return u32be(#Data) .. Name .. Data .. u32be(crc32(Name .. Data))
end

local function zlibStore(Data)
    local Parts = { string.char(120, 1) }
    local Index = 1

    while Index <= #Data do
        local Length = math.min(65535, #Data - Index + 1)
        local Final = Index + Length - 1 >= #Data and 1 or 0
        local Inverse = 65535 - Length

        Parts[#Parts + 1] = string.char(Final)
        Parts[#Parts + 1] = string.char(
            bit32.band(Length, 255),
            bit32.band(bit32.rshift(Length, 8), 255)
        )
        Parts[#Parts + 1] = string.char(
            bit32.band(Inverse, 255),
            bit32.band(bit32.rshift(Inverse, 8), 255)
        )
        Parts[#Parts + 1] = Data:sub(Index, Index + Length - 1)

        Index += Length
    end

    Parts[#Parts + 1] = u32be(adler32(Data))
    return table.concat(Parts)
end

local function captureAssetPng(AssetId, Size)
    AssetId = tonumber(AssetId)
    Size = tonumber(Size) or 64

    if not AssetId or AssetId <= 0 then
        return nil
    end

    local AssetService = game:GetService("AssetService")
    local EditableImage

    local ContentValue
    pcall(function()
        ContentValue = Content.fromAssetId(AssetId)
    end)

    local Created = pcall(function()
        EditableImage = AssetService:CreateEditableImageAsync(
            ContentValue or ("rbxassetid://" .. tostring(AssetId))
        )
    end)

    if not Created or not EditableImage then
        return nil
    end

    local Success, Result = pcall(function()
        local SourceSize = EditableImage.Size
        local Width = math.floor(SourceSize.X)
        local Height = math.floor(SourceSize.Y)

        if Width < 1 or Height < 1 then
            return nil
        end

        local Pixels = EditableImage:ReadPixelsBuffer(Vector2.zero, SourceSize)
        local Rows = {}

        for Y = 0, Size - 1 do
            local SourceY = math.floor(Y * Height / Size)
            local Row = { "\0" }

            for X = 0, Size - 1 do
                local SourceX = math.floor(X * Width / Size)
                local Offset = (SourceY * Width + SourceX) * 4

                Row[#Row + 1] = string.char(
                    buffer.readu8(Pixels, Offset),
                    buffer.readu8(Pixels, Offset + 1),
                    buffer.readu8(Pixels, Offset + 2),
                    buffer.readu8(Pixels, Offset + 3)
                )
            end

            Rows[#Rows + 1] = table.concat(Row)
        end

        local Raw = table.concat(Rows)
        local Header = u32be(Size) .. u32be(Size) .. string.char(8, 6, 0, 0, 0)

        return "\137PNG\r\n\26\n"
            .. pngChunk("IHDR", Header)
            .. pngChunk("IDAT", zlibStore(Raw))
            .. pngChunk("IEND", "")
    end)

    pcall(function()
        EditableImage:Destroy()
    end)

    return Success and Result or nil
end

local IconQueue = {}
local IconQueued = {}
local IconUploaded = {}
local IconAttempts = {}
local IconWorkerRunning = false
local IconDone = 0
local IconFailed = 0

local function queueIcon(AssetId)
    if not ICON_SYNC then
        return
    end

    AssetId = tonumber(AssetId)

    if not AssetId or AssetId <= 0 or IconUploaded[AssetId] or IconQueued[AssetId] then
        return
    end

    if (IconAttempts[AssetId] or 0) >= 2 then
        return
    end

    IconQueued[AssetId] = true
    IconQueue[#IconQueue + 1] = AssetId
end

local function uploadExactIcon(AssetId)
    local Png = captureAssetPng(AssetId, ICON_SIZE)

    if not Png then
        return false
    end

    local Encoded = base64Encode(Png)

    local Success, Response = pcall(function()
        return Request({
            Url = ICON_UPLOAD_URL,
            Method = "POST",
            Headers = {
                ["Content-Type"] = "application/json",
                ["X-API-Key"] = API_KEY
            },
            Body = HttpService:JSONEncode({
                assetId = AssetId,
                png = Encoded
            }),
            Timeout = 20
        })
    end)

    if not Success then
        return false
    end

    local StatusCode = tonumber(Response.StatusCode or Response.Status) or 0
    return StatusCode >= 200 and StatusCode < 300
end

local function startIconWorker()
    if IconWorkerRunning or not ICON_SYNC then
        return
    end

    IconWorkerRunning = true

    task.spawn(function()
        while Running do
            local AssetId = table.remove(IconQueue, 1)

            if not AssetId then
                break
            end

            IconQueued[AssetId] = nil
            IconAttempts[AssetId] = (IconAttempts[AssetId] or 0) + 1

            if uploadExactIcon(AssetId) then
                IconUploaded[AssetId] = true
                IconDone += 1
            else
                IconFailed += 1
            end

            task.wait(ICON_SYNC_DELAY)
        end

        IconWorkerRunning = false

        if IconDone > 0 or IconFailed > 0 then
            print(("SEASHOP: exact icon cache | uploaded %d | failed %d"):format(IconDone, IconFailed))
        end
    end)
end

local function queueSnapshotIcons(Payload)
    if not ICON_SYNC or type(Payload) ~= "table" then
        return
    end

    for _, Item in ipairs(Payload.pets or {}) do
        queueIcon(Item.petIconAssetId or Item.iconAssetId)
    end

    for _, List in ipairs({
        Payload.eggs or {},
        Payload.fieldEggs or {},
        Payload.stolenEggs or {}
    }) do
        for _, Item in ipairs(List) do
            queueIcon(Item.eggIconAssetId or Item.iconAssetId)
        end
    end

    startIconWorker()
end

local DebuggedCategories = {}
local DebugIconCount = 0

local function describeIcon(Value)
    local ValueType = typeof(Value)
    local Text = ""

    local Success = pcall(function()
        Text = tostring(Value)
    end)

    if not Success then
        Text = "<tostring failed>"
    end

    if #Text > 220 then
        Text = Text:sub(1, 220) .. "..."
    end

    return ValueType, Text
end

local function debugAssetIcons(Category, Data, Egg, ParsedPetId, ParsedEggId)
    if not DEBUG_ICONS or DebuggedCategories[Category] or DebugIconCount >= DEBUG_ICON_LIMIT then
        return
    end

    DebuggedCategories[Category] = true
    DebugIconCount += 1

    local PetType, PetRaw = describeIcon(Data and Data.Icon)
    local EggType, EggRaw = describeIcon(Egg and Egg.Icon)

    print(("[SEASHOP ICON DEBUG] category=%s display=%s"):format(
        tostring(Category),
        tostring(Data and Data.DisplayName or "?")
    ))
    print(("  petIconType=%s parsedId=%s raw=%s"):format(
        PetType,
        tostring(ParsedPetId),
        PetRaw
    ))
    print(("  eggIconType=%s parsedId=%s raw=%s"):format(
        EggType,
        tostring(ParsedEggId),
        EggRaw
    ))

    if ParsedPetId == 0 and PetRaw ~= "" then
        warn("[SEASHOP ICON DEBUG] pet icon exists but parser returned 0:", tostring(Category), PetRaw)
    end

    if ParsedEggId == 0 and EggRaw ~= "" then
        warn("[SEASHOP ICON DEBUG] egg icon exists but parser returned 0:", tostring(Category), EggRaw)
    end

    if (PetRaw == "" or PetRaw == "nil") and (EggRaw == "" or EggRaw == "nil") then
        warn("[SEASHOP ICON DEBUG] NO ICON FOUND IN ASSETS:", tostring(Category))
    end
end

local function getAssetInfo(Category)
    local Data = Assets[Category]

    if type(Data) ~= "table" then
        return {
            name = tostring(Category or "Unknown"),
            eggName = tostring(Category or "Unknown") .. " Egg",
            rarity = "Common",
            rarityNumber = 1,
            rarityColor = "",
            income = 0,
            icon = 0,
            eggIcon = 0,
            iconRaw = "",
            eggIconRaw = "",
            growthTime = 120
        }
    end

    local Rarity = Data.Rarity
    local RarityId = "Common"
    local RarityName = "Common"
    local RarityNumber = 1
    local RarityColor = ""

    if type(Rarity) == "table" then
        RarityId = tostring(Rarity._id or Rarity.DisplayName or "Common")
        RarityName = tostring(Rarity.DisplayName or RarityId)
        RarityNumber = tonumber(Rarity.RarityNumber) or 1
        RarityColor = colorToHex(Rarity.Color)
    elseif Rarity ~= nil then
        RarityId = tostring(Rarity)
        RarityName = RarityId
    end

    local RarityData = RarityById[RarityId]

    if type(RarityData) == "table" then
        RarityName = tostring(RarityData.DisplayName or RarityName)
        RarityNumber = tonumber(RarityData.RarityNumber) or RarityNumber

        if RarityColor == "" then
            RarityColor = colorToHex(RarityData.Color)
        end
    end

    local Egg = type(Data.Egg) == "table" and Data.Egg or {}
    local PetIconId = getAssetId(Data.Icon)
    local EggIconId = getAssetId(Egg.Icon)

    debugAssetIcons(Category, Data, Egg, PetIconId, EggIconId)

    return {
        name = tostring(Data.DisplayName or Category),
        eggName = tostring(Egg.DisplayName or ((Data.DisplayName or Category) .. " Egg")),
        rarity = RarityName,
        rarityNumber = RarityNumber,
        rarityColor = RarityColor,
        income = tonumber(Data.EarningRate) or 0,
        icon = PetIconId,
        eggIcon = EggIconId,
        iconRaw = getRawIcon(Data.Icon),
        eggIconRaw = getRawIcon(Egg.Icon),
        growthTime = tonumber(Egg.GrowthTime) or 120
    }
end

local function getMutations(Value)
    local Result = {}
    local Seen = {}

    local function add(Id)
        if Id == nil then
            return
        end

        local Raw = tostring(Id)
        if Raw == "" or Seen[Raw] then
            return
        end

        if Mutations and Raw == tostring(Mutations.NO_MUTATION) then
            return
        end

        local Name = Raw

        if Mutations and type(Mutations.LabelOf) == "function" then
            pcall(function()
                Name = Mutations.LabelOf(Id) or Raw
            end)
        end

        Name = tostring(Name)

        if not Seen[Name] then
            Seen[Name] = true
            table.insert(Result, Name)
        end
    end

    if type(Value) == "table" then
        local Count = 0

        for Key, Entry in pairs(Value) do
            Count += 1
            if Count > 12 then
                break
            end

            if type(Key) == "number" then
                add(Entry)
            elseif Entry == true then
                add(Key)
            else
                add(Entry)
            end
        end
    else
        add(Value)
    end

    table.sort(Result)
    return Result
end

local function getEggWeight(Category, Scale, Record)
    local Weight = tonumber(Record and (Record.Weight or Record.WeightKg)) or 0

    if AssetItems and type(AssetItems.GetWeightKgForScale) == "function" then
        pcall(function()
            Weight = tonumber(AssetItems.GetWeightKgForScale(Category, tonumber(Scale))) or Weight
        end)
    end

    return Weight
end

local function getPetWeight(Record)
    local Weight = tonumber(Record and (Record.Weight or Record.WeightKg)) or 0

    if AssetItems
        and type(AssetItems.Decode) == "function"
        and type(AssetItems.WeightKg) == "function"
    then
        pcall(function()
            Weight = tonumber(AssetItems.WeightKg(AssetItems.Decode(Record))) or Weight
        end)
    end

    return Weight
end

local function getSaveData()
    if not Save or type(Save.Get) ~= "function" then
        return {}
    end

    local Success, Data = pcall(Save.Get)
    return Success and type(Data) == "table" and Data or {}
end

local function readPets(Data)
    local Pets = {}
    local Equipped = {}
    local TotalIncome = 0

    for _, Uid in ipairs(type(Data.EquippedAssets) == "table" and Data.EquippedAssets or {}) do
        Equipped[tostring(Uid)] = true
    end

    for Uid, Record in pairs(type(Data.Inventory) == "table" and Data.Inventory or {}) do
        if type(Record) == "table" and Record.Category then
            local Category = tostring(Record.Category)
            local Info = getAssetInfo(Category)
            local Income = tonumber(Record.Income or Record.EarningRate or Record.Rate) or Info.income

            TotalIncome += Income

            table.insert(Pets, {
                uid = tostring(Uid),
                category = Category,
                name = Info.name,
                eggName = Info.eggName,
                hatchName = Info.name,
                rarity = Info.rarity,
                rarityNumber = Info.rarityNumber,
                rarityColor = Info.rarityColor,
                weight = getPetWeight(Record),
                income = Income,
                baseIncome = Info.income,
                mutations = getMutations(Record.Mutations or Record.Mutation),
                iconAssetId = Info.icon,
                eggIconAssetId = Info.eggIcon,
                petIconAssetId = Info.icon,
                icon = Info.iconRaw,
                eggIcon = Info.eggIconRaw,
                petIcon = Info.iconRaw,
                equipped = Equipped[tostring(Uid)] == true,
                favorite = Record.IsFavorite == true,
                inFuse = Record.InFuse == true,
                source = "pet"
            })
        end
    end

    table.sort(Pets, function(A, B)
        if A.rarityNumber == B.rarityNumber then
            return A.income > B.income
        end

        return A.rarityNumber > B.rarityNumber
    end)

    return Pets, TotalIncome
end

local function readEggs(Data)
    local Eggs = {}

    for Uid, Record in pairs(type(Data.EggInventory) == "table" and Data.EggInventory or {}) do
        if type(Record) == "table" and Record.AssetCategory then
            local Category = tostring(Record.AssetCategory)
            local Info = getAssetInfo(Category)
            local Placed = type(Record.Placement) == "table"
            local GrowthTime = Info.growthTime
            local Speed = tonumber(Record.GrowthSpeedMultiplier) or 1

            if Speed > 0 then
                GrowthTime /= Speed
            end

            local Remaining = 0

            if Placed then
                local PlacedAt = tonumber(Record.Placement.PlacedAt)

                if PlacedAt then
                    Remaining = math.max(
                        0,
                        PlacedAt + GrowthTime - Workspace:GetServerTimeNow()
                    )
                end
            end

            table.insert(Eggs, {
                uid = tostring(Uid),
                category = Category,
                name = Info.eggName,
                eggName = Info.eggName,
                hatchName = Info.name,
                rarity = Info.rarity,
                rarityNumber = Info.rarityNumber,
                rarityColor = Info.rarityColor,
                weight = getEggWeight(Category, Record.AssetScale, Record),
                income = tonumber(Record.Income or Record.EarningRate) or Info.income,
                baseIncome = Info.income,
                mutations = getMutations(Record.Mutations or Record.Mutation),
                iconAssetId = Info.eggIcon,
                eggIconAssetId = Info.eggIcon,
                petIconAssetId = Info.icon,
                icon = Info.eggIconRaw,
                eggIcon = Info.eggIconRaw,
                petIcon = Info.iconRaw,
                placed = Placed,
                growthRemaining = Remaining,
                growthTotal = GrowthTime,
                area = tostring(Record.AreaId or Record.Area or ""),
                state = Placed and "Growing" or "Bag",
                source = "egg"
            })
        end
    end

    table.sort(Eggs, function(A, B)
        if A.placed ~= B.placed then
            return A.placed
        end

        if A.rarityNumber == B.rarityNumber then
            return A.weight > B.weight
        end

        return A.rarityNumber > B.rarityNumber
    end)

    return Eggs
end

local function readFieldEggs()
    local Eggs = {}

    if not EggState or type(EggState.ReadFieldEggs) ~= "function" then
        return Eggs
    end

    pcall(function()
        local State = EggState.ReadFieldEggs()

        for _, Record in ipairs(type(State) == "table" and State.Records or {}) do
            if type(Record) == "table" then
                local Category = Record.AssetCategory or Record.Category

                if Category then
                    Category = tostring(Category)

                    local Info = getAssetInfo(Category)

                    table.insert(Eggs, {
                        uid = tostring(Record.Uid or Record.UID or ""),
                        category = Category,
                        name = Info.eggName,
                        eggName = Info.eggName,
                        hatchName = Info.name,
                        rarity = tostring(Record.Rarity or Info.rarity),
                        rarityNumber = tonumber(Record.Rank or Record.RarityRank) or Info.rarityNumber,
                        rarityColor = Info.rarityColor,
                        weight = getEggWeight(Category, Record.AssetScale or Record.Scale, Record),
                        income = tonumber(Record.Income or Record.EarningRate) or Info.income,
                        baseIncome = Info.income,
                        mutations = getMutations(Record.Mutations or Record.Mutation),
                        iconAssetId = Info.eggIcon,
                        eggIconAssetId = Info.eggIcon,
                        petIconAssetId = Info.icon,
                        icon = Info.eggIconRaw,
                        eggIcon = Info.eggIconRaw,
                        petIcon = Info.iconRaw,
                        area = tostring(Record.AreaId or Record.Area or ""),
                        state = tostring(Record.State or "Slot"),
                        carrierUserId = tonumber(Record.CarrierUserId or Record.Carrier) or 0,
                        source = "field"
                    })
                end
            end
        end
    end)

    return Eggs
end

local SeenInventory
local RecentField = {}
local StolenHistory = {}

local function updateStolenHistory(Eggs, FieldEggs)
    local Now = os.time() * 1000
    local Current = {}

    for _, Egg in ipairs(FieldEggs) do
        if Egg.uid ~= "" then
            RecentField[Egg.uid] = {
                time = os.clock(),
                data = Egg
            }
        end
    end

    for Uid, Entry in pairs(RecentField) do
        if os.clock() - Entry.time > 180 then
            RecentField[Uid] = nil
        end
    end

    for _, Egg in ipairs(Eggs) do
        Current[Egg.uid] = true

        if SeenInventory and not SeenInventory[Egg.uid] then
            local History = table.clone(Egg)
            local Previous = RecentField[Egg.uid]

            History.timestamp = Now
            History.source = Previous and "field" or "inventory"

            if Previous and History.area == "" then
                History.area = Previous.data.area
            end

            table.insert(StolenHistory, 1, History)

            while #StolenHistory > 100 do
                table.remove(StolenHistory)
            end
        end
    end

    SeenInventory = Current
end

local function buildSnapshot()
    local Data = getSaveData()
    local Pets, PetIncome = readPets(Data)
    local Eggs = readEggs(Data)
    local FieldEggs = readFieldEggs()

    updateStolenHistory(Eggs, FieldEggs)

    local DirectIncome = tonumber(
        Data.MoneyPerSecond
        or Data.IncomePerSecond
        or Data.EarningsPerSecond
        or Data.EarningRate
    )

    local Humanoid = Player.Character and Player.Character:FindFirstChildOfClass("Humanoid")

    local BossTokens =
        tonumber(Data.BossTokens)
        or (type(Data.Currencies) == "table" and tonumber(Data.Currencies.BossTokens))
        or (type(Data.Currency) == "table" and tonumber(Data.Currency.BossTokens))
        or 0

    return {
        username = Player.Name,
        displayName = Player.DisplayName,
        userId = Player.UserId,
        placeId = game.PlaceId,
        jobId = game.JobId,
        gameName = "Steal An Egg",
        placeName = "Steal An Egg",
        session = Session,

        stats = {
            money = tonumber(Data.Money) or 0,
            moneyPerSecond = DirectIncome or PetIncome,
            incomeSource = DirectIncome and "save" or "pet-base-sum",
            speedPower = tonumber(Data.SpeedPower) or 0,
            walkSpeed = Humanoid and Humanoid.WalkSpeed or 0,
            bossTokens = BossTokens,
            treadmillLevel = tonumber(Data.TreadmillUpgradeLevel) or 0,
            eggCount = #Eggs,
            petCount = #Pets
        },

        pets = Pets,
        eggs = Eggs,
        fieldEggs = FieldEggs,
        stolenEggs = StolenHistory,
        catalogVersion = "runtime",
        ts = os.time() * 1000
    }
end

local function debugSnapshotIcons(Payload)
    if not DEBUG_ICONS then
        return
    end

    local Missing = {}
    local Seen = {}

    local function check(List, Kind)
        for _, Item in ipairs(List or {}) do
            local Raw = tostring(Item.icon or Item.petIcon or Item.eggIcon or "")
            local Id = tonumber(Item.iconAssetId or Item.petIconAssetId or Item.eggIconAssetId) or 0

            if Id == 0 and (Raw == "" or Raw == "nil") then
                local Key = Kind .. ":" .. tostring(Item.category)
                if not Seen[Key] then
                    Seen[Key] = true
                    table.insert(Missing, {
                        kind = Kind,
                        category = tostring(Item.category),
                        name = tostring(Item.name)
                    })
                end
            end
        end
    end

    check(Payload.pets, "pet")
    check(Payload.eggs, "egg")
    check(Payload.fieldEggs, "field")

    print(("[SEASHOP ICON DEBUG] snapshot pets=%d eggs=%d field=%d unresolvedCategories=%d"):format(
        #(Payload.pets or {}),
        #(Payload.eggs or {}),
        #(Payload.fieldEggs or {}),
        #Missing
    ))

    for Index, Item in ipairs(Missing) do
        if Index > 30 then
            warn("[SEASHOP ICON DEBUG] more unresolved categories omitted:", #Missing - 30)
            break
        end
        warn(("[SEASHOP ICON DEBUG] unresolved %s category=%s name=%s"):format(
            Item.kind,
            Item.category,
            Item.name
        ))
    end
end

local function sendSnapshot()
    if not SEND_DATA then
        return false
    end

    local Payload = buildSnapshot()
    queueSnapshotIcons(Payload)
    debugSnapshotIcons(Payload)

    local Success, Response = pcall(function()
        return Request({
            Url = API_URL,
            Method = "POST",
            Headers = {
                ["Content-Type"] = "application/json",
                ["X-API-Key"] = API_KEY
            },
            Body = HttpService:JSONEncode(Payload),
            Timeout = 15
        })
    end)

    if not Success then
        warn("SEASHOP: request failed:", Response)
        return false
    end

    local StatusCode = tonumber(Response.StatusCode or Response.Status) or 0

    if StatusCode >= 200 and StatusCode < 300 then
        print(
            ("SEASHOP: synced | pets %d | eggs %d | field %d")
            :format(#Payload.pets, #Payload.eggs, #Payload.fieldEggs)
        )
        return true
    end

    local Body = tostring(Response.Body or "")
    warn("SEASHOP: web error:", StatusCode, Body)

    if StatusCode == 503 and Body:find("NOPERM", 1, true) then
        pcall(function()
            StarterGui:SetCore("SendNotification", {
                Title = "SEASHOP Steal An Egg",
                Text = "Redis write denied. Replace UPSTASH_REDIS_REST_TOKEN in Vercel, then redeploy.",
                Duration = 7
            })
        end)
    end

    return false
end

local Connected = sendSnapshot()

pcall(function()
    StarterGui:SetCore("SendNotification", {
        Title = "SEASHOP Steal An Egg",
        Text = Connected and "Web sync connected" or "Web sync failed - check console",
        Duration = 5
    })
end)

task.spawn(function()
    while Running and Player.Parent do
        task.wait(SEND_INTERVAL)

        if Running and SEND_DATA then
            sendSnapshot()
        end
    end
end)

print("SEASHOP: Steal An Egg Tracker v" .. TRACKER_VERSION .. " loaded")
return Env.SEASHOP_SAE_TRACKER_STOP
