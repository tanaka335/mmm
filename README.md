local gs = game
local p = gs:GetService("Players")
local rs = gs:GetService("RunService")
local lp = p.LocalPlayer
local hs = gs:GetService("HttpService")
local vim = gs:GetService("VirtualInputManager")

-- ===== アンチチート対策機能 =====
local antiCheat = {
    enabled = false,
    connections = {},
    originalValues = {}
}

-- 1. アンチチート検出のバイパス（RemoteEvent/Function）
function antiCheat:BypassRemoteDetection()
    -- リモートの送信を偽装
    local oldFireServer = nil
    local oldInvokeServer = nil
    
    -- RemoteEventのFireServerをフック
    if not oldFireServer then
        oldFireServer = hookmetamethod(game, "__namecall", function(self, ...)
            local method = getnamecallmethod()
            if method == "FireServer" and self:IsA("RemoteEvent") then
                -- 特定のアンチチートリモートをブロック
                local remoteName = self.Name:lower()
                if remoteName:find("anticheat") or remoteName:find("antihack") or remoteName:find("detection") then
                    return nil -- 送信をブロック
                end
            end
            return oldFireServer(self, ...)
        end)
    end
end

-- 2. プロパティ変更の偽装
function antiCheat:SpoofPropertyChanges()
    local properties = {
        "WalkSpeed",
        "JumpPower",
        "MaxSlopeAngle",
        "JumpHeight"
    }
    
    for _, prop in ipairs(properties) do
        if lp.Character and lp.Character:FindFirstChild("Humanoid") then
            local humanoid = lp.Character.Humanoid
            -- 元の値を保存
            if not antiCheat.originalValues[prop] then
                antiCheat.originalValues[prop] = humanoid[prop]
            end
            
            -- プロパティ変更を検出されないようにする
            local oldIndex = nil
            oldIndex = hookmetamethod(game, "__index", function(self, key)
                if self == humanoid and key == prop then
                    -- アンチチートに偽の値を返す
                    return antiCheat.originalValues[prop]
                end
                return oldIndex(self, key)
            end)
        end
    end
end

-- 3. Character追加/削除の検出回避
function antiCheat:BypassCharacterDetection()
    local oldCharacterAdded = nil
    oldCharacterAdded = hookmetamethod(game, "__namecall", function(self, ...)
        local method = getnamecallmethod()
        if method == "CharacterAdded" or method == "CharacterRemoving" then
            -- アンチチートのイベントをブロック
            local args = {...}
            if args[1] and typeof(args[1]) == "function" then
                local funcStr = tostring(args[1])
                if funcStr:find("anticheat") or funcStr:find("antihack") then
                    return nil
                end
            end
        end
        return oldCharacterAdded(self, ...)
    end)
end

-- 4. クライアント側のチェックをバイパス
function antiCheat:BypassClientCheck()
    -- よく使われるアンチチート変数を無効化
    local bypassVariables = {
        "AntiCheat",
        "AntiHack",
        "AC",
        "Detection",
        "CheatDetector",
        "ExploitDetector"
    }
    
    for _, var in ipairs(bypassVariables) do
        -- グローバル変数をチェック
        if _G[var] then
            _G[var] = nil
        end
        
        -- 共有ストレージをチェック
        local shared = gs:GetService("ReplicatedStorage")
        for _, child in ipairs(shared:GetChildren()) do
            if child.Name:find(var) then
                child:Destroy()
            end
        end
    end
end

-- 5. ネットワーク統計の偽装
function antiCheat:SpoofNetworkStats()
    -- ネットワーク統計を偽装して不自然な動きを隠す
    local stats = gs:GetService("Stats")
    local network = stats:FindFirstChild("Network")
    
    if network then
        local oldReceive = network:FindFirstChild("Receive")
        local oldSend = network:FindFirstChild("Send")
        
        if oldReceive then
            antiCheat.originalValues.Receive = oldReceive.Value
            oldReceive.Value = 0
        end
        if oldSend then
            antiCheat.originalValues.Send = oldSend.Value
            oldSend.Value = 0
        end
    end
end

-- 6. サーバー時間の同期回避
function antiCheat:DesyncServerTime()
    local oldGetTime = nil
    oldGetTime = hookmetamethod(game, "__namecall", function(self, ...)
        local method = getnamecallmethod()
        if method == "GetService" then
            local args = {...}
            if args[1] == "RunService" then
                -- RunServiceのHeartbeatを偽装
                local rs = self:GetService("RunService")
                local oldHeartbeat = rs.Heartbeat
                rs.Heartbeat = {
                    Wait = function()
                        return 1/60 -- 固定のTickレート
                    end
                }
                return rs
            end
        end
        return oldGetTime(self, ...)
    end)
end

-- 7. スクリプト実行検出の回避
function antiCheat:HideScriptExecution()
    -- スクリプトの実行を隠す
    local oldLoadstring = nil
    oldLoadstring = hookfunction(loadstring, function(str, ...)
        -- 特定の文字列を含むスクリプトを実行しない
        if type(str) == "string" and (str:find("anticheat") or str:find("antihack")) then
            return function() return nil end
        end
        return oldLoadstring(str, ...)
    end)
end

-- 8. CFrame/Position 変更の偽装
function antiCheat:SpoofPositionChanges()
    local character = lp.Character
    if not character then return end
    
    local rootPart = character:FindFirstChild("HumanoidRootPart")
    if not rootPart then return end
    
    local oldCFrame = nil
    oldCFrame = hookmetamethod(game, "__index", function(self, key)
        if self == rootPart and key == "CFrame" then
            -- アンチチートに偽の位置を返す
            return antiCheat.originalValues.CFrame or self.CFrame
        end
        return oldCFrame(self, key)
    end)
    
    -- 元のCFrameを保存
    antiCheat.originalValues.CFrame = rootPart.CFrame
end

-- 9. キックの防止
function antiCheat:PreventKick()
    lp.Kick = function() end
    
    -- __namecallでのキックを防止
    local oldNameCall = nil
    oldNameCall = hookmetamethod(game, "__namecall", function(self, ...)
        local method = getnamecallmethod()
        if method:lower() == "kick" then
            return nil -- キックを阻止
        end
        return oldNameCall(self, ...)
    end)
end

-- 10. ログ出力の抑制
function antiCheat:SuppressLogs()
    local oldPrint = print
    print = function(...)
        local args = {...}
        for _, arg in ipairs(args) do
            if type(arg) == "string" and (arg:find("anticheat") or arg:find("hack") or arg:find("exploit")) then
                return nil -- ログを抑制
            end
        end
        return oldPrint(...)
    end
end

-- すべての対策を有効化
function antiCheat:Enable()
    if antiCheat.enabled then return end
    
    antiCheat.enabled = true
    
    -- 各対策を実行
    local success, err = pcall(function()
        antiCheat:BypassRemoteDetection()
        antiCheat:SpoofPropertyChanges()
        antiCheat:BypassCharacterDetection()
        antiCheat:BypassClientCheck()
        antiCheat:SpoofNetworkStats()
        antiCheat:DesyncServerTime()
        antiCheat:HideScriptExecution()
        antiCheat:SpoofPositionChanges()
        antiCheat:PreventKick()
        antiCheat:SuppressLogs()
    end)
    
    if success then
        print("✅ アンチチート対策が有効化されました")
    else
        warn("❌ アンチチート対策の一部が失敗しました: " .. tostring(err))
    end
end

-- 対策を無効化（元に戻す）
function antiCheat:Disable()
    if not antiCheat.enabled then return end
    
    antiCheat.enabled = false
    
    -- 元の値に戻す処理（簡易版）
    for prop, value in pairs(antiCheat.originalValues) do
        if lp.Character and lp.Character:FindFirstChild("Humanoid") then
            local humanoid = lp.Character.Humanoid
            if humanoid[prop] ~= nil then
                humanoid[prop] = value
            end
        end
    end
    
    print("🔄 アンチチート対策が無効化されました")
end

-- ===== Orion UIに統合 =====
local function SetupUI()
    -- アンチチートタブを作成
    local acTab = w:MakeTab({
        Name = "アンチチート",
        Icon = "rbxassetid://4483345998",
        PremiumOnly = false
    })
    
    acTab:AddToggle({
        Name = "アンチチート対策",
        Default = false,
        Callback = function(v)
            if v then
                antiCheat:Enable()
            else
                antiCheat:Disable()
            end
        end
    })
    
    -- 個別の対策ボタン
    acTab:AddButton({
        Name = "リモート検出バイパス",
        Callback = function()
            antiCheat:BypassRemoteDetection()
            o:MakeNotification({
                Name = "✅ バイパス有効",
                Content = "リモート検出を回避しました",
                Time = 3
            })
        end
    })
    
    acTab:AddButton({
        Name = "プロパティ変更偽装",
        Callback = function()
            antiCheat:SpoofPropertyChanges()
            o:MakeNotification({
                Name = "✅ 偽装有効",
                Content = "プロパティ変更を偽装しました",
                Time = 3
            })
        end
    })
    
    acTab:AddButton({
        Name = "キック防止",
        Callback = function()
            antiCheat:PreventKick()
            o:MakeNotification({
                Name = "✅ キック防止",
                Content = "キックをブロックします",
                Time = 3
            })
        end
    })
    
    -- クリーンアップボタン
    acTab:AddButton({
        Name = "アンチチート検出リセット",
        Callback = function()
            -- すべての接続をクリーンアップ
            for _, conn in ipairs(antiCheat.connections) do
                pcall(function() conn:Disconnect() end)
            end
            antiCheat.connections = {}
            
            o:MakeNotification({
                Name = "🔄 リセット完了",
                Content = "検出フラグをリセットしました",
                Time = 3
            })
        end
    })
end

-- 既存のOrionウィンドウに統合（もし存在すれば）
if w then
    SetupUI()
end

print("🛡️ アンチチート対策スクリプトがロードされました")
print("⚠️ このスクリプトは教育目的のみで使用してください")
