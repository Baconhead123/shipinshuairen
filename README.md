-- Uhhhhhh Reanimate - PhysicsRepRootPart 无碰撞箱甩飞系统
-- 仅供教育研究目的

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
local LocalPlayer = Players.LocalPlayer
local Character = LocalPlayer.Character or LocalPlayer.CharacterAdded:Wait()

-- 配置参数
local Settings = {
    FlingPower = 16384,          -- 甩飞力量
    FlingHeight = 5,             -- 初始高度
    TargetDistance = 50,         -- 目标检测距离
    UsePrediction = true,        -- 使用预测算法
    PredictionTime = 0.2,        -- 预测时间（秒）
    RotationSpeed = 500,         -- 旋转速度
    AutoTarget = true,           -- 自动选择目标
    TargetSpecificPlayer = "",   -- 指定目标玩家名称
    UseHatFling = false,         -- 使用帽子模式（需要帽子）
    Cooldown = 1,                -- 冷却时间（秒）
}

-- 工具类函数
local Utils = {}

-- 预测算法
function Utils.PredictPosition(targetPart, timeOffset)
    if not targetPart or not targetPart:IsDescendantOf(Workspace) then
        return targetPart and targetPart.Position or Vector3.new()
    end
    
    local currentPos = targetPart.Position
    local velocity = targetPart.AssemblyLinearVelocity or Vector3.new()
    local gravity = Workspace.Gravity
    
    -- 基础预测：位置 = 当前位置 + 速度 * 时间
    local predictedPos = currentPos + velocity * timeOffset
    
    -- 考虑重力影响
    predictedPos = predictedPos + Vector3.new(0, -gravity * 0.5 * timeOffset * timeOffset, 0)
    
    -- 添加随机扰动避免过于规律
    local randomOffset = Vector3.new(
        math.random(-10, 10) * 0.01,
        math.random(-5, 5) * 0.01,
        math.random(-10, 10) * 0.01
    )
    
    return predictedPos + randomOffset
end

-- 获取最近玩家
function Utils.GetNearestPlayer(maxDistance)
    local nearestPlayer = nil
    local nearestDistance = maxDistance or Settings.TargetDistance
    local localRoot = Character:FindFirstChild("HumanoidRootPart")
    
    if not localRoot then return nil end
    
    for _, player in pairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character then
            local targetRoot = player.Character:FindFirstChild("HumanoidRootPart")
            if targetRoot then
                local distance = (localRoot.Position - targetRoot.Position).Magnitude
                if distance < nearestDistance then
                    nearestDistance = distance
                    nearestPlayer = player
                end
            end
        end
    end
    
    return nearestPlayer
end

-- 获取指定玩家
function Utils.GetSpecificPlayer(name)
    for _, player in pairs(Players:GetPlayers()) do
        if player.Name:lower():find(name:lower()) then
            return player
        end
    end
    return nil
end

-- 主甩飞类
local PhysicsFling = {}

function PhysicsFling:Initialize()
    self.RootPart = Character:WaitForChild("HumanoidRootPart")
    self.Humanoid = Character:WaitForChild("Humanoid")
    self.LastFlingTime = 0
    
    -- 如果是帽子模式，准备帽子
    if Settings.UseHatFling and Character:FindFirstChildOfClass("Accessory") then
        self.HatHandle = Character:FindFirstChildOfClass("Accessory").Handle
    else
        self.HatHandle = nil
    end
    
    print("[PhysicsFling] 初始化完成 - 模式:", Settings.UseHatFling and "帽子模式" or "肢体模式")
end

-- 核心甩飞函数
function PhysicsFling:ExecuteFling(targetPlayer)
    if tick() - self.LastFlingTime < Settings.Cooldown then
        return
    end
    
    if not targetPlayer or not targetPlayer.Character then
        warn("[PhysicsFling] 无效目标")
        return
    end
    
    local targetRoot = targetPlayer.Character:FindFirstChild("HumanoidRootPart")
    if not targetRoot then
        warn("[PhysicsFling] 目标没有HumanoidRootPart")
        return
    end
    
    -- 计算预测位置
    local targetPosition = targetRoot.Position
    if Settings.UsePrediction then
        targetPosition = Utils.PredictPosition(targetRoot, Settings.PredictionTime)
    end
    
    -- 计算甩飞的CFrame（在目标上方）
    local flingCFrame = CFrame.new(
        targetPosition + Vector3.new(0, Settings.FlingHeight, 0)
    ) * CFrame.Angles(
        math.rad(math.random(0, 360)),
        math.rad(math.random(0, 360)),
        math.rad(math.random(0, 360))
    )
    
    -- 计算甩飞速度
    local flingVelocity = Vector3.new(
        math.random(-100, 100),
        -Settings.FlingPower,  -- 主要向下力
        math.random(-100, 100)
    )
    
    local flingRotVelocity = Vector3.new(
        Settings.RotationSpeed,
        Settings.RotationSpeed,
        Settings.RotationSpeed
    )
    
    -- 核心：设置PhysicsRepRootPart并应用力
    if Settings.UseHatFling and self.HatHandle then
        -- 帽子模式
        pcall(function()
            -- 设置PhysicsRepRootPart为目标
            self.HatHandle:SetAttribute("PhysicsRepRootPart", targetRoot)
            
            -- 对帽子施加力
            self.HatHandle.AssemblyLinearVelocity = flingVelocity
            self.HatHandle.AssemblyAngularVelocity = flingRotVelocity
            self.HatHandle.CFrame = flingCFrame
            
            print("[PhysicsFling] 帽子甩飞执行于:", targetPlayer.Name)
        end)
    else
        -- 肢体模式（直接控制自己的RootPart）
        pcall(function()
            -- 关键步骤：建立物理同步链路
            self.RootPart:SetAttribute("PhysicsRepRootPart", targetRoot)
            
            -- 对自己RootPart施加剧烈运动
            self.RootPart.CFrame = flingCFrame
            self.RootPart.Velocity = flingVelocity
            self.RootPart.RotVelocity = flingRotVelocity
            
            print("[PhysicsFling] 肢体甩飞执行于:", targetPlayer.Name)
        end)
    end
    
    self.LastFlingTime = tick()
    
    -- 短暂延迟后清除PhysicsRepRootPart（避免持续控制）
    task.delay(0.1, function()
        pcall(function()
            if Settings.UseHatFling and self.HatHandle then
                self.HatHandle:SetAttribute("PhysicsRepRootPart", nil)
            else
                self.RootPart:SetAttribute("PhysicsRepRootPart", nil)
            end
        end)
    end)
end

-- 自动甩飞循环
function PhysicsFling:StartAutoFling()
    self.AutoFlingConnection = RunService.Heartbeat:Connect(function()
        local target = nil
        
        if Settings.TargetSpecificPlayer ~= "" then
            target = Utils.GetSpecificPlayer(Settings.TargetSpecificPlayer)
        elseif Settings.AutoTarget then
            target = Utils.GetNearestPlayer()
        end
        
        if target then
            self:ExecuteFling(target)
        end
    end)
    
    print("[PhysicsFling] 自动甩飞已启动")
end

function PhysicsFling:Stop()
    if self.AutoFlingConnection then
        self.AutoFlingConnection:Disconnect()
        self.AutoFlingConnection = nil
    end
    
    -- 清理PhysicsRepRootPart
    pcall(function()
        if self.RootPart then
            self.RootPart:SetAttribute("PhysicsRepRootPart", nil)
        end
        if self.HatHandle then
            self.HatHandle:SetAttribute("PhysicsRepRootPart", nil)
        end
    end)
    
    print("[PhysicsFling] 已停止")
end

-- UI控制界面
local function CreateControlUI()
    local ScreenGui = Instance.new("ScreenGui")
    ScreenGui.Name = "PhysicsFlingUI"
    ScreenGui.Parent = game.CoreGui
    
    local MainFrame = Instance.new("Frame")
    MainFrame.Size = UDim2.new(0, 300, 0, 400)
    MainFrame.Position = UDim2.new(0, 10, 0, 10)
    MainFrame.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
    MainFrame.BorderSizePixel = 2
    MainFrame.BorderColor3 = Color3.fromRGB(60, 60, 60)
    MainFrame.Parent = ScreenGui
    
    local Title = Instance.new("TextLabel")
    Title.Text = "PhysicsRepRootPart 甩飞系统"
    Title.Size = UDim2.new(1, 0, 0, 30)
    Title.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
    Title.TextColor3 = Color3.fromRGB(255, 100, 100)
    Title.Font = Enum.Font.SourceSansBold
    Title.Parent = MainFrame
    
    -- 配置选项
    local yOffset = 40
    local function CreateOption(labelText, valueName, defaultValue, isString)
        local frame = Instance.new("Frame")
        frame.Size = UDim2.new(1, -20, 0, 25)
        frame.Position = UDim2.new(0, 10, 0, yOffset)
        frame.BackgroundTransparency = 1
        frame.Parent = MainFrame
        
        local label = Instance.new("TextLabel")
        label.Text = labelText
        label.Size = UDim2.new(0.6, 0, 1, 0)
        label.TextColor3 = Color3.fromRGB(200, 200, 200)
        label.TextXAlignment = Enum.TextXAlignment.Left
        label.BackgroundTransparency = 1
        label.Parent = frame
        
        local textBox = Instance.new("TextBox")
        textBox.Size = UDim2.new(0.4, 0, 1, 0)
        textBox.Position = UDim2.new(0.6, 0, 0, 0)
        textBox.Text = tostring(defaultValue)
        textBox.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
        textBox.TextColor3 = Color3.fromRGB(255, 255, 255)
        textBox.Parent = frame
        
        textBox.FocusLost:Connect(function()
            if isString then
                Settings[valueName] = textBox.Text
            else
                Settings[valueName] = tonumber(textBox.Text) or defaultValue
            end
        end)
        
        yOffset = yOffset + 30
    end
    
    -- 创建配置项
    CreateOption("甩飞力量:", "FlingPower", 16384)
    CreateOption("目标距离:", "TargetDistance", 50)
    CreateOption("预测时间:", "PredictionTime", 0.2)
    CreateOption("指定玩家:", "TargetSpecificPlayer", "")
    
    -- 开关选项
    local function CreateToggle(labelText, valueName, defaultValue)
        local frame = Instance.new("Frame")
        frame.Size = UDim2.new(1, -20, 0, 25)
        frame.Position = UDim2.new(0, 10, 0, yOffset)
        frame.BackgroundTransparency = 1
        frame.Parent = MainFrame
        
        local label = Instance.new("TextLabel")
        label.Text = labelText
        label.Size = UDim2.new(0.7, 0, 1, 0)
        label.TextColor3 = Color3.fromRGB(200, 200, 200)
        label.TextXAlignment = Enum.TextXAlignment.Left
        label.BackgroundTransparency = 1
        label.Parent = frame
        
        local toggle = Instance.new("TextButton")
        toggle.Size = UDim2.new(0.3, 0, 1, 0)
        toggle.Position = UDim2.new(0.7, 0, 0, 0)
        toggle.Text = defaultValue and "ON" or "OFF"
        toggle.BackgroundColor3 = defaultValue and Color3.fromRGB(0, 170, 0) or Color3.fromRGB(170, 0, 0)
        toggle.TextColor3 = Color3.fromRGB(255, 255, 255)
        toggle.Parent = frame
        
        toggle.MouseButton1Click:Connect(function()
            Settings[valueName] = not Settings[valueName]
            toggle.Text = Settings[valueName] and "ON" or "OFF"
            toggle.BackgroundColor3 = Settings[valueName] and Color3.fromRGB(0, 170, 0) or Color3.fromRGB(170, 0, 0)
        end)
        
        yOffset = yOffset + 30
    end
    
    CreateToggle("自动目标:", "AutoTarget", true)
    CreateToggle("使用预测:", "UsePrediction", true)
    CreateToggle("帽子模式:", "UseHatFling", false)
    
    -- 控制按钮
    local StartButton = Instance.new("TextButton")
    StartButton.Text = "启动自动甩飞"
    StartButton.Size = UDim2.new(1, -20, 0, 30)
    StartButton.Position = UDim2.new(0, 10, 0, yOffset + 10)
    StartButton.BackgroundColor3 = Color3.fromRGB(0, 120, 0)
    StartButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    StartButton.Parent = MainFrame
    
    local StopButton = Instance.new("TextButton")
    StopButton.Text = "停止"
    StopButton.Size = UDim2.new(1, -20, 0, 30)
    StopButton.Position = UDim2.new(0, 10, 0, yOffset + 50)
    StopButton.BackgroundColor3 = Color3.fromRGB(120, 0, 0)
    StopButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    StopButton.Parent = MainFrame
    
    -- 单次甩飞按钮
    local SingleFlingButton = Instance.new("TextButton")
    SingleFlingButton.Text = "甩飞最近玩家"
    SingleFlingButton.Size = UDim2.new(1, -20, 0, 30)
    SingleFlingButton.Position = UDim2.new(0, 10, 0, yOffset + 90)
    SingleFlingButton.BackgroundColor3 = Color3.fromRGB(0, 0, 120)
    SingleFlingButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    SingleFlingButton.Parent = MainFrame
    
    return ScreenGui, StartButton, StopButton, SingleFlingButton
end

-- 主执行函数
local function Main()
    -- 初始化甩飞系统
    PhysicsFling:Initialize()
    
    -- 创建控制界面
    local UI, StartBtn, StopBtn, SingleBtn = CreateControlUI()
    
    -- 按钮事件
    StartBtn.MouseButton1Click:Connect(function()
        PhysicsFling:StartAutoFling()
    end)
    
    StopBtn.MouseButton1Click:Connect(function()
        PhysicsFling:Stop()
    end)
    
    SingleBtn.MouseButton1Click:Connect(function()
        local target = Utils.GetNearestPlayer()
        if target then
            PhysicsFling:ExecuteFling(target)
        end
    end)
    
    -- 角色死亡时清理
    Character.Humanoid.Died:Connect(function()
        PhysicsFling:Stop()
    end)
    
    print("=== PhysicsRepRootPart Fling System Loaded ===")
    print("警告：此脚本仅供教育目的")
    print("使用可能导致账户封禁！")
    
    -- 脚本卸载时清理
    game:GetService("UserInputService").InputBegan:Connect(function(input, processed)
        if input.KeyCode == Enum.KeyCode.P and not processed then
            PhysicsFling:Stop()
            UI:Destroy()
            print("脚本已卸载")
        end
    end)
end

-- 等待角色加载后执行
if Character then
    Main()
else
    LocalPlayer.CharacterAdded:Wait()
    Character = LocalPlayer.Character
    Main()
end

-- 键盘快捷键说明
print("按 P 键卸载脚本")
print("UI已显示在屏幕左上角")
