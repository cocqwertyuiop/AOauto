local Fluent = loadstring(game:HttpGet("https://github.com/StyearX/Fluent-modded/releases/download/1.5.1/FluentPro"))()

local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")

local Window = Fluent:CreateWindow({
    Title = "塔防助手",
    SubTitle = "Tower Defense Helper",
    Version = "v1.2",
    TabWidth = 130,
    Size = UDim2.fromOffset(480, 520),
    Acrylic = true,
    Theme = "Amoled",
    MinimizeKey = Enum.KeyCode.RightControl,
    Search = true,
    Icons = "solar/planet-bold",
})

local Tabs = {
    Main = Window:AddTab({ Title = "主功能", Icon = "solar/home-bold" }),
    Auto = Window:AddTab({ Title = "一键执行", Icon = "solar/magic-wand-bold" }),
    Settings = Window:AddTab({ Title = "设置", Icon = "solar/settings-bold" }),
}

local function Notify(title, content, ntype, icon, duration)
    Fluent:Notify({
        Title = title,
        Content = content,
        Type = ntype or "Info",
        Icon = icon,
        Duration = duration or 3,
    })
end

-- ==================== 兑换 Code ====================
local secCode = Tabs.Main:AddSection("兑换 Code", "solar/code-bold")

local isRedeeming = false
secCode:AddButton({
    Title = "一键兑换所有 Code",
    Icon = "solar/gift-bold",
    Description = "点一次即可，包含：THANKYOU!, Release!, TYKaito!, Origins, AO, GAMESPEED",
    Callback = function()
        if isRedeeming then
            Notify("兑换中", "请稍候，正在兑换...", "Warning", "solar/clock-circle-bold", 3)
            return
        end
        isRedeeming = true
        
        task.spawn(function()
            local Event = game:GetService("ReplicatedStorage").LobbyRemotes.CodesFunction
            local codes = {"THANKYOU!", "Release!", "TYKaito!", "Origins", "AO", "GAMESPEED"}
            
            Notify("兑换开始", "正在兑换所有代码，请耐心等待...", "Info", "solar/clock-circle-bold", 5)
            
            for i, code in ipairs(codes) do
                for attempt = 1, 3 do
                    pcall(function()
                        Event:InvokeServer("RedeemCode", code)
                    end)
                    task.wait(1)
                end
                task.wait(1)
            end
            
            Notify("兑换完成", "所有代码兑换完成！", "Success", "solar/check-circle-bold", 5)
            isRedeeming = false
        end)
    end,
})

-- ==================== 领取奖励 ====================
local secClaim = Tabs.Main:AddSection("领取奖励", "solar/gift-bold")

secClaim:AddButton({
    Title = "领取所有奖励",
    Icon = "solar/medal-bold",
    Description = "领取战斗通行证 + 每日奖励",
    Callback = function()
        task.spawn(function()
            pcall(function()
                local BattlepassEvent = game:GetService("ReplicatedStorage").LobbyRemotes.BattlepassRemote
                BattlepassEvent:FireServer("ClaimBattlepass", "Season1")
            end)
            task.wait(1)
            pcall(function()
                local DailyEvent = game:GetService("ReplicatedStorage").Remotes.RemoteEvent
                DailyEvent:FireServer("ClaimDailyReward", "Normal")
            end)
            Notify("领取完成", "所有奖励已领取！", "Success", "solar/check-circle-bold", 5)
        end)
    end,
})

-- ==================== 抽奖设置 ====================
local secSummonSettings = Tabs.Main:AddSection("抽奖设置", "solar/settings-bold")

secSummonSettings:AddButton({
    Title = "开启设置（跳过动画 + 最大抽奖）",
    Icon = "solar/fast-forward-bold",
    Description = "跳过抽奖动画并开启最大抽奖数",
    Callback = function()
        task.spawn(function()
            pcall(function()
                local SettingsEvent = game:GetService("ReplicatedStorage").Remotes.SettingsRemote
                SettingsEvent:FireServer("SetSetting", "SkipSummonCutscene", true)
                task.wait(0.5)
                SettingsEvent:FireServer("SetSetting", "SummonMax", true)
            end)
            Notify("设置完成", "跳过动画 + 最大抽奖数已开启", "Success", "solar/check-circle-bold", 5)
        end)
    end,
})

-- ==================== 抽奖 ====================
local secSummon = Tabs.Main:AddSection("抽奖", "solar/star-bold")

local summoning = false
local summonToggle = secSummon:AddToggle("SummonToggle", {
    Title = "自动抽奖",
    Icon = "solar/star-bold",
    Default = false,
    Description = "10连抽，失败自动降级为单抽，间隔2秒",
    Callback = function(state)
        summoning = state
        
        if state then
            task.spawn(function()
                local SummonEvent = game:GetService("ReplicatedStorage").LobbyRemotes.SummonFunction
                
                while summoning do
                    local pulled = false
                    
                    local ok10, result10 = pcall(function()
                        return SummonEvent:InvokeServer("SummonTower", "Standard", 10)
                    end)
                    
                    if ok10 and result10 then
                        Notify("抽奖成功", "10连抽完成！", "Success", "solar/star-bold", 3)
                        pulled = true
                    end
                    
                    if not pulled and summoning then
                        task.wait(1)
                        
                        local ok1, result1 = pcall(function()
                            return SummonEvent:InvokeServer("SummonTower", "Standard", 1)
                        end)
                        
                        if ok1 and result1 then
                            Notify("抽奖成功", "单抽完成（10连失败降级）", "Success", "solar/star-bold", 3)
                            pulled = true
                        end
                    end
                    
                    if not pulled then
                        Notify("抽奖失败", "钻石可能不足，自动停止", "Warning", "solar/danger-triangle-bold", 5)
                        summoning = false
                        summonToggle:SetValue(false)
                        break
                    end
                    
                    task.wait(2)
                end
            end)
        else
            Notify("停止抽奖", "自动抽奖已停止", "Info", "solar/pause-circle-bold", 3)
        end
    end,
})

-- ==================== 自动售卖 ====================
local secAutoSell = Tabs.Main:AddSection("自动售卖", "solar/cart-bold")

local function SendAutoSell(rarity, enabled)
    local SummonRemote = game:GetService("ReplicatedStorage").LobbyRemotes.SummonRemote
    pcall(function()
        SummonRemote:FireServer("ToggleAutoSell", "Standard", rarity)
    end)
    Notify("自动售卖", rarity .. " 自动售卖: " .. (enabled and "开启" or "关闭"), 
        enabled and "Success" or "Info", "solar/cart-bold", 3)
end

local RareSellToggle = secAutoSell:AddToggle("RareSell", {
    Title = "Rare 自动售卖",
    Icon = "solar/star-bold",
    Default = false,
    Description = "自动售卖 Rare 品质塔",
    Callback = function(v)
        SendAutoSell("Rare", v)
    end,
})

local LegendarySellToggle = secAutoSell:AddToggle("LegendarySell", {
    Title = "Legendary 自动售卖",
    Icon = "solar/star-bold",
    Default = false,
    Description = "自动售卖 Legendary 品质塔",
    Callback = function(v)
        SendAutoSell("Legendary", v)
    end,
})

local EpicSellToggle = secAutoSell:AddToggle("EpicSell", {
    Title = "Epic 自动售卖",
    Icon = "solar/star-bold",
    Default = false,
    Description = "自动售卖 Epic 品质塔",
    Callback = function(v)
        SendAutoSell("Epic", v)
    end,
})

-- ==================== 特征跳过 ====================
local secTrait = Tabs.Main:AddSection("特征跳过", "solar/shield-star-bold")

local TraitSkipToggle = secTrait:AddToggle("TraitSkip", {
    Title = "关闭所有特征（排除皇冠）",
    Icon = "solar/shield-star-bold",
    Default = false,
    Description = "跳过 Looting, Hustler, Ascendant, Decay, Stride, Rupture, Joker, Overseer",
    Callback = function(v)
        task.spawn(function()
            local TraitRemote = game:GetService("ReplicatedStorage").Remotes.TraitRerollRemote
            local traits = {"Looting", "Hustler", "Ascendant", "Decay", "Stride", "Rupture", "Joker", "Overseer"}
            
            for _, trait in ipairs(traits) do
                pcall(function()
                    TraitRemote:FireServer("ToggleTraitSkip", trait)
                end)
                task.wait(0.3)
            end
            
            Notify("特征跳过", v and "已开启所有特征跳过（排除皇冠）" or "已关闭所有特征跳过", 
                v and "Success" or "Info", "solar/shield-star-bold", 5)
        end)
    end,
})

-- ==================== 一键执行 ====================
local secAuto = Tabs.Auto:AddSection("一键执行", "solar/magic-wand-bold")

secAuto:AddParagraph({
    Title = "一键执行所有功能",
    Content = "按顺序自动执行：\n1. 兑换所有 Code\n2. 领取所有奖励\n3. 开启抽奖设置\n4. 开启自动售卖\n5. 开启自动抽奖\n6. 关闭所有特征（排除皇冠）\n\n请确保你在私人服务器中，且已有足够钻石",
})

local isAutoRunning = false

secAuto:AddButton({
    Title = "🚀 开始一键执行",
    Icon = "solar/magic-wand-bold",
    Description = "按顺序自动执行所有功能",
    Callback = function()
        if isAutoRunning then
            Notify("执行中", "一键执行正在进行中，请稍候...", "Warning", "solar/clock-circle-bold", 3)
            return
        end
        isAutoRunning = true
        
        task.spawn(function()
            local step = 0
            
            -- 步骤 1：兑换所有 Code
            step += 1
            Notify("步骤 1/6", "开始兑换所有 Code...", "Info", "solar/code-bold", 5)
            
            local Event = game:GetService("ReplicatedStorage").LobbyRemotes.CodesFunction
            local codes = {"THANKYOU!", "Release!", "TYKaito!", "Origins", "AO", "GAMESPEED"}
            
            for i, code in ipairs(codes) do
                for attempt = 1, 3 do
                    pcall(function()
                        Event:InvokeServer("RedeemCode", code)
                    end)
                    task.wait(0.5)
                end
                task.wait(0.5)
            end
            
            Notify("步骤 1/6 完成", "所有 Code 兑换完成！", "Success", "solar/check-circle-bold", 4)
            task.wait(1)
            
            -- 步骤 2：领取所有奖励
            step += 1
            Notify("步骤 2/6", "开始领取所有奖励...", "Info", "solar/gift-bold", 5)
            
            pcall(function()
                local BattlepassEvent = game:GetService("ReplicatedStorage").LobbyRemotes.BattlepassRemote
                BattlepassEvent:FireServer("ClaimBattlepass", "Season1")
            end)
            task.wait(1)
            pcall(function()
                local DailyEvent = game:GetService("ReplicatedStorage").Remotes.RemoteEvent
                DailyEvent:FireServer("ClaimDailyReward", "Normal")
            end)
            
            Notify("步骤 2/6 完成", "所有奖励领取完成！", "Success", "solar/check-circle-bold", 4)
            task.wait(1)
            
            -- 步骤 3：开启抽奖设置
            step += 1
            Notify("步骤 3/6", "开启抽奖设置（跳过动画 + 最大抽奖）...", "Info", "solar/settings-bold", 5)
            
            pcall(function()
                local SettingsEvent = game:GetService("ReplicatedStorage").Remotes.SettingsRemote
                SettingsEvent:FireServer("SetSetting", "SkipSummonCutscene", true)
                task.wait(0.5)
                SettingsEvent:FireServer("SetSetting", "SummonMax", true)
            end)
            
            Notify("步骤 3/6 完成", "抽奖设置已开启！", "Success", "solar/check-circle-bold", 4)
            task.wait(1)
            
            -- 步骤 4：开启自动售卖
            step += 1
            Notify("步骤 4/6", "开启自动售卖（Rare + Legendary + Epic）...", "Info", "solar/cart-bold", 5)
            
            local SummonRemote = game:GetService("ReplicatedStorage").LobbyRemotes.SummonRemote
            local sellRarities = {"Rare", "Legendary", "Epic"}
            
            for _, rarity in ipairs(sellRarities) do
                pcall(function()
                    SummonRemote:FireServer("ToggleAutoSell", "Standard", rarity)
                end)
                task.wait(0.5)
            end
            
            -- 同步 Toggle 状态
            RareSellToggle:SetValue(true)
            LegendarySellToggle:SetValue(true)
            EpicSellToggle:SetValue(true)
            
            Notify("步骤 4/6 完成", "自动售卖已开启！", "Success", "solar/check-circle-bold", 4)
            task.wait(1)
            
            -- 步骤 5：开启自动抽奖
            step += 1
            Notify("步骤 5/6", "开启自动抽奖...", "Info", "solar/star-bold", 5)
            
            summoning = true
            summonToggle:SetValue(true)
            
            task.spawn(function()
                local SummonEvent = game:GetService("ReplicatedStorage").LobbyRemotes.SummonFunction
                
                while summoning do
                    local pulled = false
                    
                    local ok10, result10 = pcall(function()
                        return SummonEvent:InvokeServer("SummonTower", "Standard", 10)
                    end)
                    
                    if ok10 and result10 then
                        pulled = true
                    end
                    
                    if not pulled and summoning then
                        task.wait(1)
                        
                        local ok1, result1 = pcall(function()
                            return SummonEvent:InvokeServer("SummonTower", "Standard", 1)
                        end)
                        
                        if ok1 and result1 then
                            pulled = true
                        end
                    end
                    
                    if not pulled then
                        summoning = false
                        summonToggle:SetValue(false)
                        break
                    end
                    
                    task.wait(2)
                end
            end)
            
            Notify("步骤 5/6 完成", "自动抽奖已开启！", "Success", "solar/check-circle-bold", 4)
            task.wait(1)
            
            -- 步骤 6：关闭所有特征
            step += 1
            Notify("步骤 6/6", "关闭所有特征（排除皇冠）...", "Info", "solar/shield-star-bold", 5)
            
            local TraitRemote = game:GetService("ReplicatedStorage").Remotes.TraitRerollRemote
            local traits = {"Looting", "Hustler", "Ascendant", "Decay", "Stride", "Rupture", "Joker", "Overseer"}
            
            for _, trait in ipairs(traits) do
                pcall(function()
                    TraitRemote:FireServer("ToggleTraitSkip", trait)
                end)
                task.wait(0.3)
            end
            
            TraitSkipToggle:SetValue(true)
            
            Notify("步骤 6/6 完成", "所有特征已关闭（排除皇冠）！", "Success", "solar/check-circle-bold", 4)
            task.wait(1)
            
            Notify("🎉 全部完成", "一键执行所有功能已完成！", "Success", "solar/party-bold", 6)
            isAutoRunning = false
        end)
    end,
})

secAuto:AddDivider()

secAuto:AddParagraph({
    Title = "注意事项",
    Content = "• 一键执行需要较长时间（约30-60秒）\n• 执行过程中请勿重复点击\n• 自动抽奖需要足够钻石\n• 钻石不足时自动抽奖会停止\n• 所有功能只在私人服务器测试使用",
})

-- ==================== 设置 Tab ====================
local secAbout = Tabs.Settings:AddSection("关于", "solar/info-circle-bold")

secAbout:AddParagraph({
    Title = "塔防助手 v1.2",
    Content = "私人服务器测试使用\n所有功能点一次即可\n请勿影响他人游戏体验",
})

local InterfaceManager = loadstring(game:HttpGet("https://raw.githubusercontent.com/StyearX/Fluent-modded/main/InterfaceManager.lua"))()
local SaveManager = loadstring(game:HttpGet("https://raw.githubusercontent.com/StyearX/Fluent-modded/main/SaveManager.lua"))()

InterfaceManager:SetLibrary(Fluent)
InterfaceManager:SetFolder("TowerDefenseHelper")
InterfaceManager:BuildInterfaceSection(Tabs.Settings)
InterfaceManager:LoadSettings()

SaveManager:SetLibrary(Fluent)
SaveManager:SetFolder("TowerDefenseHelper/Config")
SaveManager:IgnoreThemeSettings()
SaveManager:BuildConfigSection(Tabs.Settings)
SaveManager:LoadAutoloadConfig()

Notify("塔防助手", "脚本加载成功！", "Success", "solar/check-circle-bold", 5)

Window:SelectTab(1)
