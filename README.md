# aftermathEsp
-- Aftermath Pro Sigma Script by Antgt_
-- Fixed helper aliasing and string concatenation

local Rayfield = loadstring(game:HttpGet('https://[Log in to view URL]'))()

local Window = Rayfield:CreateWindow({
   Name = "Aftermath Pro Sigma Script",
   LoadingTitle = "Aftermath Script",
   LoadingSubtitle = "by Antgt_,",
   ConfigurationSaving = {
      Enabled = true,
      FolderName = nil,
      FileName = "Aftermath Script by Antgt_"
   },
   Discord = {
      Enabled = false,
      Invite = "noinvitelink",
      RememberJoins = true
   },
   KeySystem = true,
   KeySettings = {
      Title = "Key",
      Subtitle = "Key is Hello",
      Note = "Thanks for using!",
      FileName = "Key Aftermath Antgt_",
      SaveKey = true,
      GrabKeyFromSite = false,
      Key = {"Hello"}
   }
})

local MainTab = Window:CreateTab("Visuals", nil)
local Section = MainTab:CreateSection("visuals")

local Button = MainTab:CreateButton({
   Name = "Player ESP",
   Callback = function()
       -- Alias helpers
       local global  = getgenv().global
       local declare = global.declare
       local get     = global.get

       -- ESP Variables
       local FillColor        = Color3.fromRGB(0, 138, 142)
       local DepthMode        = "AlwaysOnTop"
       local FillTransparency = 0.5
       local OutlineColor     = Color3.fromRGB(255, 255, 255)
       local OutlineTransparency = 0
       local CoreGui          = game:GetService("CoreGui")
       local Players          = game:GetService("Players")
       local connections      = {}

       local Storage = Instance.new("Folder")
       Storage.Parent = CoreGui
       Storage.Name   = "Highlight_Storage"

       local players = cloneref(Players)
       local client  = players.LocalPlayer
       local camera  = workspace.CurrentCamera

       -- Helper: loop manager
       declare(global, "services", {})
       function global.declare(self, index, value, check)
           if self[index] == nil then
               self[index] = value
           elseif check then
               for _, method in ipairs({"remove", "Disconnect"}) do
                   pcall(function() value[method](value) end)
               end
           end
           return self[index]
       end

       function global.get(service)
           return services[service]
       end

       declare(services, "loop", {})
       declare(services, "new", {})
       declare(get("loop"), "cache", {})

       -- Loop runner
       get("loop").new = function(self, index, func, disabled)
           if disabled == nil and (func == nil or typeof(func) == "boolean") then
               disabled = func
               func = index
           end
           self.cache[index] = {
               enabled = not disabled,
               func    = func,
               toggle  = function(self, flag)
                   self.enabled = (flag == nil) and not self.enabled or flag
               end,
               remove  = function() self.cache[index] = nil end
           }
           return self.cache[index]
       end

       declare(get("loop"), "connection",
           cloneref(game:GetService("RunService")).RenderStepped:Connect(function(delta)
               for _, loop in pairs(get("loop").cache) do
                   if loop.enabled then
                       local ok, err = pcall(function() loop.func(delta) end)
                       if not ok then warn(err) end
                   end
               end
           end), true)

       -- Drawing factory
       get("new").drawing = function(class, props)
           local d = Drawing.new(class)
           for prop, val in pairs(props) do pcall(function() d[prop] = val end) end
           return d
       end

       -- Player cache
       declare(get("player"), "cache", {})
       get("player").find = function(self, pl)
           for char, data in pairs(self.cache) do
               if data.player == pl then return char end
           end
       end

       get("player").check = function(self, pl)
           local ok, res = pcall(function()
               local char = pl:IsA("Player") and pl.Character or pl
               local root = char:WaitForChild('HumanoidRootPart', 5)
               return root and char.Parent ~= nil
           end)
           return ok and res
       end

       get("player").new = function(self, pl)
           local function cacheCharacter(char)
               self.cache[char] = { player = pl, drawings = {}, highlight = nil }
               local d = self.cache[char].drawings
               d.box        = get("new").drawing("Square", {Visible=false})
               d.boxFilled  = get("new").drawing("Square", {Visible=false, Filled=true})
               d.boxOutline = get("new").drawing("Square", {Visible=false})
               d.name       = get("new").drawing("Text",   {Visible=false, Center=true})
               d.distance   = get("new").drawing("Text",   {Visible=false, Center=true})

               -- Highlight instance
               local hl = Instance.new("Highlight")
               hl.Name              = pl.Name
               hl.FillColor         = FillColor
               hl.DepthMode         = DepthMode
               hl.FillTransparency  = FillTransparency
               hl.OutlineColor      = OutlineColor
               hl.OutlineTransparency = OutlineTransparency
               hl.Parent            = Storage
               hl.Adornee           = char
               self.cache[char].highlight = hl

               connections[pl] = pl.CharacterAdded:Connect(function(c) hl.Adornee = c end)
           end

           local function watch(char)
               if self:check(pl) then cacheCharacter(char)
               else
                   local lst = char.ChildAdded:Connect(function()
                       if self:check(pl) then cacheCharacter(char); lst:Disconnect() end
                   end)
               end
           end

           if pl.Character then watch(pl.Character) end
           pl.CharacterAdded:Connect(watch)
       end

       -- Remove player visuals
       get("player").remove = function(self, pl)
           local char = self:find(pl)
           if char then
               local data = self.cache[char]
               for _, d in pairs(data.drawings) do d:Remove() end
               if data.highlight then data.highlight:Destroy() end
               if connections[pl] then connections[pl]:Disconnect() end
               self.cache[char] = nil
           end
       end

       -- Update loop
       declare(get("player"), "loop",
           get("loop").new(function()
               for char, data in pairs(get("player").cache) do
                   get("player").update(get("player"), char, data)
               end
           end), true)

       -- Update per-frame
       get("player").update = function(self, char, data)
           if not self:check(data.player) then return self:remove(data.player) end
           local root = char:WaitForChild('HumanoidRootPart')
           local dist = (client.Character.HumanoidRootPart.Position - root.Position).Magnitude
           data.distance = dist

           task.spawn(function()
               local pos, vis = camera:WorldToViewportPoint(root.Position)
               local visOK = vis and data.distance <= get("visuals").renderDistance
                               and (not get("visuals").teamCheck or data.player.Team ~= client.Team)
               local scale = 1/(pos.Z * math.tan(math.rad(camera.FieldOfView*0.5))*2)*1000
               local w,h = math.floor(4.5*scale), math.floor(6*scale)
               local x,y = math.floor(pos.X), math.floor(pos.Y)
               local x0,y0 = x-w/2, y-h/2 + (0.5*scale)
               local d = data.drawings

               if visOK then
                   d.box.Size          = Vector2.new(w,h)
                   d.box.Position      = Vector2.new(x0,y0)
                   d.box.Color         = get("boxes").color
                   d.box.Thickness     = 1

                   d.boxFilled.Size        = d.box.Size
                   d.boxFilled.Position    = d.box.Position
                   d.boxFilled.Color       = get("boxes").filled.color
                   d.boxFilled.Transparency= get("boxes").filled.transparency

                   d.boxOutline.Size       = d.box.Size
                   d.boxOutline.Position   = d.box.Position
                   d.boxOutline.Color      = get("boxes").outline.color
                   d.boxOutline.Thickness  = 3

                   d.name.Text           = "[ " .. data.player.Name .. " ]"
                   d.name.Position       = Vector2.new(x, y0 - d.name.TextBounds.Y - 2)
                   d.name.Outline        = get("names").outline.enabled
                   d.name.OutlineColor   = get("names").outline.color
                   d.name.Size           = math.clamp(12.5*scale, 10,12.5)

                   d.distance.Text       = "[ " .. math.floor(dist) .. " ]"
                   d.distance.Position   = Vector2.new(x, y0 + h + (d.distance.TextBounds.Y*0.25))
                   d.distance.Outline    = get("distance").outline.enabled
                   d.distance.OutlineColor = get("distance").outline.color
                   d.distance.Size       = math.clamp(11*scale, 10,11)
               end

               d.box.Visible      = visOK and get("boxes").enabled
               d.boxFilled.Visible= d.box.Visible and get("boxes").filled.enabled
               d.boxOutline.Visible= d.box.Visible and get("boxes").outline.enabled
               d.name.Visible     = visOK and get("names").enabled
               d.distance.Visible = visOK and get("distance").enabled
           end)
       end

       -- Default features config
       declare(global, "features", {})
       features = global.features
       features.visuals = {
           enabled = true,
           teamCheck = false,
           renderDistance = 4000,
           boxes = {enabled=true, color=Color3.new(1,1,1), outline={enabled=true,color=Color3.new(239/255,235/255,0)}, filled={enabled=false,color=Color3.new(1,1,1),transparency=0.25}},
           names = {enabled=true, outline={enabled=true,color=Color3.new(0,0,0)}},
           distance = {enabled=true, outline={enabled=true,color=Color3.new(0,0,0)}}
       }

       -- Initialize existing players
       for _, pl in ipairs(Players:GetPlayers()) do
           if pl ~= client and not get("player"):find(pl) then get("player"):new(pl) end
       end

       -- Listen for joins/leaves
       declare(get('player'), 'added', workspace.ChildAdded:Connect(function(c)
           local pl = Players:FindFirstChild(c.Name)
           if pl and not get("player"):find(pl) then get("player"):new(pl) end
       end), true)
       declare(get('player'), 'removing', workspace.ChildRemoved:Connect(function(c)
           local pl = Players:FindFirstChild(c.Name)
           if pl then get("player"):remove(pl) end
       end), true)
   end
})
