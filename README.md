function find_library_base(lib_name, state_filter)
    local ranges = gg.getRangesList(lib_name)
    for _, range in ipairs(ranges) do
        if range.state == state_filter then
            return range.start
        end
    end
    gg.alert("Library not found or matching state not available")
    return nil
end

function isValidHex(hex) 
    return #hex > 0 and hex:lower():find("[^%dabcdef]") == nil 
end

function readMemory(offset, size)
    local ret = ""
    local function f(v) 
        v = string.format("%X", v) 
        if #v == 1 then return "0"..v end 
        return v 
    end
    for i = 0, size - 1 do 
        ret = ret..f(gg.getValues({{address = offset + i, flags = gg.TYPE_BYTE}})[1].value) 
    end
    return ret
end

function reverseHex(hex)
    local ret = ""
    if #hex == 0 then return false end
    for i = 1, #hex, 2 do 
        ret = ret..hex:sub(i, i + 1) 
    end
    return ret:upper()
end

function hex2patchTable(hex, offset)
    local ret = {}
    local i = 0
    for v in hex:gmatch("%S%S") do 
        table.insert(ret, {address = offset + i, flags = gg.TYPE_BYTE, value = v.."r"}) 
        i = i + 1 
    end
    return ret
end

function generate_modified_method_table(lib_base, method)
    local t, hex, offset = {}, method.value, method.offset
    local originalHex = reverseHex(readMemory(lib_base + offset, (#hex + #hex % 2) / 2))
    if #hex < #originalHex then
        hex = hex..originalHex:sub(#originalHex)
    end
    local hexTable = hex2patchTable(hex, lib_base + offset)
    return hexTable
end

function generate_original_method_table(lib_base, method)
    local t, hex, offset = {}, method.value, method.offset
    local originalHex = reverseHex(readMemory(lib_base + offset, (#hex + #hex % 2) / 2))
    if #hex < #originalHex then
        hex = hex..originalHex:sub(#originalHex)
    end
    local originalHexTable = hex2patchTable(originalHex, lib_base + offset)
    return originalHexTable
end

local target_info = gg.getTargetInfo()
local is64bit = target_info.x64
local arch = is64bit and "64-bit" or "32-bit"
gg.toast("Detected architecture: " .. arch)

local lib_name = "libil2cpp.so"
local lib_base = find_library_base(lib_name, "Xa")
if not lib_base then return end

local method_list_64bit = {
    {name = "Unlimited Jump", offset = 0x1b8af40, value = "E0CF8952C0035FD6"},
    {name = "Equip All Armor", offset = 0x1c83178, value = "200080D2C0035FD6"},
    {name = "Use Skill Without Weapon", offset = 0x1c722a8, value = "200080D2C0035FD6"},
    {name = "See Items Color in Chest", offset = 0x1b9389c, value = "200080D2C0035FD6"},
    {name = "Skip Search Loot", offset = 0x1c869e8, value = "200080D2C0035FD6"},
    {name = "Attack Team", offset = 0x1bae180, value = "000080D2C0035FD6"},
    {name = "Can See Rogue Stealth", offset = 0x200adec, value = "C0035FD6"},
    {name = "Increase Attack & Skill Speed", offset = 0x1ba1150, value = "400080520000271E00D8215E0000261EC0035FD6"},
}

local method_list_32bit = {
    {name = "Unlimited Jump", offset = 0x2b63984, value = "0F0702E31EFF2FE1"},
    {name = "Equip All Armor", offset = 0x2c9ad38, value = "0100A0E31EFF2FE1"},
    {name = "Use Skill Without Weapon", offset = 0x2c84bc0, value = "0100A0E31EFF2FE1"},
    {name = "See Items Color in Chest", offset = 0x2b6e30c, value = "0100A0E31EFF2FE1"},
    {name = "Skip Search Loot", offset = 0x2c9f688, value = "0100A0E31EFF2FE1"},
    {name = "Attack Team", offset = 0x2b8e77c, value = "0000A0E31EFF2FE1"},
    {name = "Can See Rogue Stealth", offset = 0x312a960, value = "1EFF2FE1"},
    {name = "Increase Attack & Skill Speed", offset = 0x2b7e4ec, value = "7A0444E31EFF2FE1"},
}

local method_list = is64bit and method_list_64bit or method_list_32bit

local method_status = {}
local original_values = {}
local modified_values = {}

for i, method in ipairs(method_list) do
    method_status[method.name] = false
    
    local method_modified_table = generate_modified_method_table(lib_base, method)
    local method_original_table = generate_original_method_table(lib_base, method)
    
    modified_values[method.name] = method_modified_table
    original_values[method.name] = method_original_table
end


function show_menu()
    local choices = {}
    local default_choices = {}

    for i, method in ipairs(method_list) do
        table.insert(choices, method.name .. (method_status[method.name] and " [🔵]" or " [🔴]"))
        table.insert(default_choices, method_status[method.name])
    end

    local selected = gg.multiChoice(choices, nil, "LuLzSeC")
    
    if selected then
        for i, _ in pairs(selected) do
            if method_status[method_list[i].name] then
                gg.setValues(original_values[method_list[i].name])
                gg.toast(method_list[i].name .. " disabled")
            else
                print(modified_values[method_list[i].name])
                gg.toast(method_list[i].name .. " enabled")
            end
            method_status[method_list[i].name] = not method_status[method_list[i].name]
        end
    else
        gg.toast("No feature selected")
    end
end

while true do
    if gg.isVisible(true) then
        gg.setVisible(false)
        show_menu()
    end
end
