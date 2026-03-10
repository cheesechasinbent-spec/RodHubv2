# RodHubv2
Nothing
local rs = game:GetService("ReplicatedStorage")
local plrs = game:GetService("Players")
local run = game:GetService("RunService")

local lp         = plrs.LocalPlayer
local ws         = game:GetService("Workspace")
local repsto     = game:GetService("ReplicatedStorage")
local stats      = game:GetService("Stats")
local uis        = game:GetService("UserInputService")
local plrgui     = lp:WaitForChild("PlayerGui")

_G.dumpedandremakedbysaturday = {
	Mode                 = "None",
	BorderKick           = false,
	MyPlot               = nil,
	StealHitbox          = nil,
	CarpetSpammedPlayers = {},
	AdminRemote          = nil,
	LastPunishTime       = {},
	TpProtector          = false,
	PlayerPositions      = {},
	TpProtectorCooldowns = {},
}

local core = _G.dumpedandremakedbysaturday

local function fireAdmin(...)
	if not core.AdminRemote then return end
	local a = {...}
	task.spawn(function() core.AdminRemote:InvokeServer(unpack(a)) end)
end

local CARPET_ITEMS = {["Flying Carpet"]=true,["Witch's Broom"]=true,["Santa's Sleigh"]=true}

function punishPlayer(p)
	if not core.AdminRemote then return end
	if not p or p==lp then return end
	local char=p.Character if not char then return end
	local hrp=char:FindFirstChild("HumanoidRootPart") if not hrp then return end
	local uid=p.UserId
	if core.LastPunishTime[uid] and tick()-core.LastPunishTime[uid]<2 then return end
	core.LastPunishTime[uid]=tick()
	hrp.CFrame=CFrame.new(0,10000,0)
	if core.Mode=="Kick" then
		fireAdmin("f888ee6e-c86d-46e1-93d7-0639d6635d42",p,"balloon")
		task.delay(0.3,function() lp:Kick("get rekt") end)
	elseif core.Mode=="NoKick" then
		fireAdmin("f888ee6e-c86d-46e1-93d7-0639d6635d42",p,"balloon")
		fireAdmin("f888ee6e-c86d-46e1-93d7-0639d6635d42",p,"ragdoll")
	end
end

local function findPlayerInHitbox()
	local hitbox=core.StealHitbox if not hitbox then return end
	local cf,size=hitbox.CFrame,hitbox.Size
	local hx,hz=size.X*0.5,size.Z*0.5
	for _,p in ipairs(plrs:GetPlayers()) do
		if p~=lp then
			local char=p.Character
			if char then
				local hrp=char:FindFirstChild("HumanoidRootPart")
				if hrp then
					local rel=cf:PointToObjectSpace(hrp.Position)
					if math.abs(rel.X)<=hx and math.abs(rel.Z)<=hz then
						for _,item in ipairs(char:GetChildren()) do
							if CARPET_ITEMS[item.Name] then punishPlayer(p) break end
						end
					end
				end
			end
		end
	end
end

task.spawn(function()
	if not lp.Character then lp.CharacterAdded:Wait() end
	task.wait(1)
	local net=repsto:WaitForChild("Packages"):WaitForChild("Net")
	local children=net:GetChildren()
	local byIdx,byName={},{}
	for i,obj in ipairs(children) do byIdx[i]=obj byName[obj.Name]=i end
	local anchorIdx=byName["RF/a0e78691-cb9b-4efc-ac08-9c06fea70059"]
	if anchorIdx then local actual=byIdx[anchorIdx+1] if actual then core.AdminRemote=actual end end
	for _,obj in ipairs(repsto:GetDescendants()) do
		if obj:IsA("RemoteEvent") then
			obj.OnClientEvent:Connect(function(...)
				if core.Mode=="None" or not core.AdminRemote or not core.MyPlot then return end
				for _,a in ipairs({...}) do
					if type(a)=="string" and a:lower():find("stealing") then
						local myHRP=lp.Character and lp.Character:FindFirstChild("HumanoidRootPart")
						if not myHRP then return end
						local best,bestDist=nil,math.huge
						for _,p in ipairs(plrs:GetPlayers()) do
							if p~=lp then
								local char=p.Character
								if char then
									local hrp=char:FindFirstChild("HumanoidRootPart")
									if hrp then
										local dist=(hrp.Position-myHRP.Position).Magnitude
										if dist<bestDist then bestDist=dist best=p end
									end
								end
							end
						end
						if best then punishPlayer(best) end
						return
					end
				end
			end)
		end
	end
end)

task.spawn(function()
	if hookfunction and fireproximityprompt then
		local old=fireproximityprompt
		hookfunction(fireproximityprompt,newcclosure(function(prompt,...)
			if core.Mode~="None" then
				local at=(prompt.ActionText or ""):lower()
				local ot=(prompt.ObjectText or ""):lower()
				if at:find("steal") or ot:find("steal") then
					local part=prompt.Parent
					if part and part:IsA("BasePart") then
						local pos=part.Position
						local best,bestD=nil,math.huge
						for _,p in ipairs(plrs:GetPlayers()) do
							if p~=lp then
								local char=p.Character
								if char then
									local hrp=char:FindFirstChild("HumanoidRootPart")
									if hrp then
										local d=(hrp.Position-pos).Magnitude
										if d<bestD then bestD=d best=p end
									end
								end
							end
						end
						if best and bestD<20 then punishPlayer(best) end
					end
					findPlayerInHitbox()
				end
			end
			return old(prompt,...)
		end))
	end
	if hookfunction and newcclosure then
		local oldFS=Instance.FireServer
		hookfunction(Instance.FireServer,newcclosure(function(self,...)
			if core.Mode~="None" and core.StealHitbox then findPlayerInHitbox() end
			return oldFS(self,...)
		end))
	end
end)

local pingLbl=nil

task.spawn(function()
	while task.wait(0.5) do
		local plots=ws:FindFirstChild("Plots")
		if plots and not core.MyPlot then
			for _,p in ipairs(plots:GetChildren()) do
				local sign=p:FindFirstChild("PlotSign")
				if sign then
					local lbl=sign:FindFirstChild("TextLabel",true)
					if lbl then
						local t=lbl.Text:lower()
						if t:find(lp.Name:lower()) or t:find(lp.DisplayName:lower()) then
							core.MyPlot=p
							core.StealHitbox=p:FindFirstChild("StealHitbox",true)
							break
						end
					end
				end
			end
		end
		if pingLbl then
			local ping=math.floor(stats.Network.ServerStatsItem["Data Ping"]:GetValue())
			local safe=ping<=150
			pingLbl.Text=(safe and "✅" or "⚠️")..ping.."ms"
			pingLbl.TextColor3=safe and Color3.fromRGB(100,255,180) or Color3.fromRGB(255,100,100)
		end
	end
end)

run.Heartbeat:Connect(function()
	if core.BorderKick and core.StealHitbox and core.AdminRemote then
		local cf,size=core.StealHitbox.CFrame,core.StealHitbox.Size
		local hx,hz=size.X*0.5,size.Z*0.5
		for _,p in ipairs(plrs:GetPlayers()) do
			if p~=lp then
				local char=p.Character
				if char then
					local hrp=char:FindFirstChild("HumanoidRootPart")
					if hrp then
						local rel=cf:PointToObjectSpace(hrp.Position)
						if math.abs(rel.X)<=hx and math.abs(rel.Z)<=hz then
							for _,item in ipairs(char:GetChildren()) do
								if CARPET_ITEMS[item.Name] then
									local uid=p.UserId
									if not core.CarpetSpammedPlayers[uid] then
										core.CarpetSpammedPlayers[uid]=true
										fireAdmin("f888ee6e-c86d-46e1-93d7-0639d6635d42",p,"balloon")
										fireAdmin("f888ee6e-c86d-46e1-93d7-0639d6635d42",p,"jumpscare")
										fireAdmin("f888ee6e-c86d-46e1-93d7-0639d6635d42",p,"rocket")
										fireAdmin("f888ee6e-c86d-46e1-93d7-0639d6635d42",p,"jail")
										task.delay(5,function() core.CarpetSpammedPlayers[uid]=nil end)
									end
									break
								end
							end
						end
					end
				end
			end
		end
	end
	if core.TpProtector and core.AdminRemote then
		for _,p in ipairs(plrs:GetPlayers()) do
			if p~=lp then
				local char=p.Character
				if char then
					local hrp=char:FindFirstChild("HumanoidRootPart")
					if hrp then
						local cur=hrp.Position
						local uid=p.UserId
						local last=core.PlayerPositions[uid]
						if last and (cur-last).Magnitude>7 then
							for _,item in ipairs(char:GetChildren()) do
								if CARPET_ITEMS[item.Name] then
									if not core.TpProtectorCooldowns[uid] or tick()-core.TpProtectorCooldowns[uid]>3 then
										core.TpProtectorCooldowns[uid]=tick()
										fireAdmin("f888ee6e-c86d-46e1-93d7-0639d6635d42",p,"balloon")
										fireAdmin("f888ee6e-c86d-46e1-93d7-0639d6635d42",p,"jail")
									end
									break
								end
							end
						end
						core.PlayerPositions[uid]=cur
					end
				end
			end
		end
	end
end)

-- ================================================================
--  TINY GALAXY UI  (110 x 155px)
-- ================================================================

local sg = Instance.new("ScreenGui")
sg.Name = "RodHubDefender"
sg.ResetOnSpawn = false
sg.DisplayOrder = 10
sg.IgnoreGuiInset = true
sg.Enabled = true
sg.Parent = plrgui

local W, H = 110, 155
local POS = UDim2.new(0.5,0,0.5,0)

local starBG = Instance.new("Frame")
starBG.AnchorPoint = Vector2.new(0.5,0.5)
starBG.Position = POS
starBG.Size = UDim2.new(0,W,0,H)
starBG.BackgroundColor3 = Color3.fromRGB(2,1,12)
starBG.BorderSizePixel = 0
starBG.ClipsDescendants = true
starBG.ZIndex = 1
starBG.Parent = sg
Instance.new("UICorner",starBG).CornerRadius = UDim.new(0,8)

local ystroke = Instance.new("UIStroke")
ystroke.Color = Color3.fromRGB(255,215,0)
ystroke.Thickness = 2
ystroke.Transparency = 0
ystroke.Parent = starBG

local neb = Instance.new("Frame")
neb.Size = UDim2.new(1,0,1,0)
neb.BackgroundTransparency = 0.55
neb.BorderSizePixel = 0
neb.ZIndex = 2
neb.Parent = starBG
local ng = Instance.new("UIGradient")
ng.Color = ColorSequence.new({
	ColorSequenceKeypoint.new(0,   Color3.fromRGB(26,0,52)),
	ColorSequenceKeypoint.new(0.5, Color3.fromRGB(2,2,18)),
	ColorSequenceKeypoint.new(1,   Color3.fromRGB(14,0,38)),
})
ng.Rotation = 120
ng.Parent = neb

local starData = {}
for i = 1, 40 do
	local sf = Instance.new("Frame")
	local sz = math.random(1,2)
	sf.Size = UDim2.new(0,sz,0,sz)
	local rx = math.random(0,10000)/10000
	local ry = math.random(0,10000)/10000
	sf.Position = UDim2.new(rx,0,ry,0)
	local b = math.random(155,255)
	sf.BackgroundColor3 = Color3.fromRGB(b,b,math.min(255,b+math.random(0,45)))
	sf.BackgroundTransparency = math.random(5,50)/100
	sf.BorderSizePixel = 0
	sf.ZIndex = 3
	Instance.new("UICorner",sf).CornerRadius = UDim.new(1,0)
	sf.Parent = starBG
	starData[i] = {
		f  = sf,
		px = rx, py = ry,
		vx = (math.random(-40,40)/40) * 0.00002,
		vy = math.random(18,55)/55    * 0.00004,
	}
end

run.Heartbeat:Connect(function(dt)
	for _,s in ipairs(starData) do
		s.px = s.px + s.vx * dt * 60
		s.py = s.py + s.vy * dt * 60
		if s.py > 1.01 then s.py = -0.01 end
		if s.px > 1.01 then s.px = -0.01 end
		if s.px < -0.01 then s.px =  1.01 end
		s.f.Position = UDim2.new(s.px,0,s.py,0)
	end
end)

local frame = Instance.new("Frame")
frame.AnchorPoint = Vector2.new(0.5,0.5)
frame.Position = POS
frame.Size = UDim2.new(0,W,0,H)
frame.BackgroundTransparency = 1
frame.BorderSizePixel = 0
frame.ClipsDescendants = true
frame.Active = true
frame.ZIndex = 5
frame.Parent = sg
Instance.new("UICorner",frame).CornerRadius = UDim.new(0,8)

local tb = Instance.new("Frame")
tb.Size = UDim2.new(1,0,0,17)
tb.BackgroundColor3 = Color3.fromRGB(4,2,18)
tb.BackgroundTransparency = 0.1
tb.BorderSizePixel = 0
tb.ZIndex = 8
tb.Parent = frame
Instance.new("UICorner",tb).CornerRadius = UDim.new(0,8)

local tbf = Instance.new("Frame")
tbf.Size = UDim2.new(1,0,0,8)
tbf.Position = UDim2.new(0,0,1,-8)
tbf.BackgroundColor3 = Color3.fromRGB(4,2,18)
tbf.BackgroundTransparency = 0.1
tbf.BorderSizePixel = 0
tbf.ZIndex = 7
tbf.Parent = tb

local tbLine = Instance.new("Frame")
tbLine.Size = UDim2.new(1,0,0,1)
tbLine.Position = UDim2.new(0,0,1,-1)
tbLine.BackgroundColor3 = Color3.fromRGB(255,215,0)
tbLine.BorderSizePixel = 0
tbLine.ZIndex = 9
tbLine.Parent = tb

local titleLbl = Instance.new("TextLabel")
titleLbl.Position = UDim2.new(0,5,0,0)
titleLbl.Size = UDim2.new(1,-40,1,0)
titleLbl.BackgroundTransparency = 1
titleLbl.Text = "✦ RodHub Defender"
titleLbl.TextColor3 = Color3.fromRGB(255,215,0)
titleLbl.TextSize = 7
titleLbl.Font = Enum.Font.GothamBold
titleLbl.TextXAlignment = Enum.TextXAlignment.Left
titleLbl.ZIndex = 10
titleLbl.Parent = tb

local pingInner = Instance.new("TextLabel")
pingInner.Position = UDim2.new(0.55,0,0.4,0)
pingInner.Size = UDim2.new(0.28,0,0.6,0)
pingInner.BackgroundTransparency = 1
pingInner.Text = "✅--ms"
pingInner.TextColor3 = Color3.fromRGB(100,255,180)
pingInner.TextSize = 5
pingInner.Font = Enum.Font.GothamBold
pingInner.ZIndex = 10
pingInner.Parent = tb
pingLbl = pingInner

local closebtn = Instance.new("TextButton")
closebtn.Position = UDim2.new(1,-13,0.5,-5)
closebtn.Size = UDim2.new(0,10,0,10)
closebtn.BackgroundColor3 = Color3.fromRGB(130,12,22)
closebtn.Text = "✕"
closebtn.TextColor3 = Color3.fromRGB(255,255,255)
closebtn.TextSize = 5
closebtn.Font = Enum.Font.GothamBold
closebtn.AutoButtonColor = false
closebtn.ZIndex = 11
closebtn.Parent = tb
Instance.new("UICorner",closebtn).CornerRadius = UDim.new(1,0)
local cst=Instance.new("UIStroke",closebtn) cst.Color=Color3.fromRGB(255,215,0) cst.Thickness=1

local minbtn = Instance.new("TextButton")
minbtn.Position = UDim2.new(1,-25,0.5,-5)
minbtn.Size = UDim2.new(0,10,0,10)
minbtn.BackgroundColor3 = Color3.fromRGB(10,15,70)
minbtn.Text = "─"
minbtn.TextColor3 = Color3.fromRGB(180,200,255)
minbtn.TextSize = 6
minbtn.Font = Enum.Font.GothamBold
minbtn.AutoButtonColor = false
minbtn.ZIndex = 11
minbtn.Parent = tb
Instance.new("UICorner",minbtn).CornerRadius = UDim.new(1,0)
local mst=Instance.new("UIStroke",minbtn) mst.Color=Color3.fromRGB(255,215,0) mst.Thickness=1

local scroll = Instance.new("ScrollingFrame")
scroll.Position = UDim2.new(0,3,0,20)
scroll.Size = UDim2.new(1,-6,1,-30)
scroll.BackgroundTransparency = 1
scroll.BorderSizePixel = 0
scroll.ClipsDescendants = true
scroll.CanvasSize = UDim2.new(0,0,0,0)
scroll.AutomaticCanvasSize = Enum.AutomaticSize.Y
scroll.ScrollBarThickness = 1
scroll.ScrollBarImageColor3 = Color3.fromRGB(255,215,0)
scroll.ScrollingDirection = Enum.ScrollingDirection.Y
scroll.ZIndex = 8
scroll.Parent = frame

local ll = Instance.new("UIListLayout")
ll.Padding = UDim.new(0,2)
ll.HorizontalAlignment = Enum.HorizontalAlignment.Center
ll.SortOrder = Enum.SortOrder.LayoutOrder
ll.Parent = scroll
Instance.new("UIPadding",scroll).PaddingTop = UDim.new(0,2)

local footer = Instance.new("Frame")
footer.AnchorPoint = Vector2.new(0,1)
footer.Position = UDim2.new(0,3,1,-2)
footer.Size = UDim2.new(1,-6,0,9)
footer.BackgroundColor3 = Color3.fromRGB(3,1,14)
footer.BackgroundTransparency = 0.2
footer.BorderSizePixel = 0
footer.ZIndex = 8
footer.Parent = frame
Instance.new("UICorner",footer).CornerRadius = UDim.new(0,4)
local fst=Instance.new("UIStroke",footer) fst.Color=Color3.fromRGB(255,215,0) fst.Thickness=1 fst.Transparency=0.4

local footerLbl = Instance.new("TextLabel")
footerLbl.Size = UDim2.new(1,0,1,0)
footerLbl.BackgroundTransparency = 1
footerLbl.Text = "✦ discord.gg/ATjcRgpU8W ✦"
footerLbl.TextColor3 = Color3.fromRGB(255,215,0)
footerLbl.TextSize = 5
footerLbl.Font = Enum.Font.GothamBold
footerLbl.TextXAlignment = Enum.TextXAlignment.Center
footerLbl.ZIndex = 9
footerLbl.Parent = footer

local tstates={Kick=false,NoKick=false,Protector=false,TpProtector=false}
local tdots,tcolors={},{}

local function setVisual(key,on)
	local d,c=tdots[key],tcolors[key]
	if d and c then
		d.Position=on and UDim2.new(1,-10,0,1) or UDim2.new(0,1,0,1)
		d.BackgroundColor3=on and c or Color3.fromRGB(35,25,55)
	end
end

local function makeRow(label,color,order,key)
	local row=Instance.new("Frame")
	row.LayoutOrder=order
	row.Size=UDim2.new(1,-2,0,17)
	row.BackgroundColor3=Color3.fromRGB(4,2,16)
	row.BackgroundTransparency=0.2
	row.BorderSizePixel=0
	row.ZIndex=9
	row.Parent=scroll
	Instance.new("UICorner",row).CornerRadius=UDim.new(0,4)
	local rs2=Instance.new("UIStroke",row)
	rs2.Color=Color3.fromRGB(255,215,0) rs2.Thickness=1 rs2.Transparency=0.45

	local acc=Instance.new("Frame")
	acc.Size=UDim2.new(0,2,0.5,0)
	acc.AnchorPoint=Vector2.new(0,0.5)
	acc.Position=UDim2.new(0,2,0.5,0)
	acc.BackgroundColor3=color
	acc.BorderSizePixel=0 acc.ZIndex=10
	acc.Parent=row
	Instance.new("UICorner",acc).CornerRadius=UDim.new(1,0)

	local lbl=Instance.new("TextLabel")
	lbl.Position=UDim2.new(0,6,0,0)
	lbl.Size=UDim2.new(0.63,0,1,0)
	lbl.BackgroundTransparency=1
	lbl.Text=label
	lbl.TextColor3=Color3.fromRGB(215,205,255)
	lbl.TextSize=6
	lbl.Font=Enum.Font.GothamBold
	lbl.TextXAlignment=Enum.TextXAlignment.Left
	lbl.ZIndex=10 lbl.Parent=row

	local track=Instance.new("TextButton")
	track.Position=UDim2.new(1,-24,0.5,-4)
	track.Size=UDim2.new(0,20,0,8)
	track.BackgroundColor3=Color3.fromRGB(8,5,24)
	track.BorderSizePixel=0 track.Text=""
	track.AutoButtonColor=false track.ZIndex=11
	track.Parent=row
	Instance.new("UICorner",track).CornerRadius=UDim.new(1,0)
	local ts2=Instance.new("UIStroke",track)
	ts2.Color=Color3.fromRGB(255,215,0) ts2.Thickness=1 ts2.Transparency=0.3

	local dot=Instance.new("Frame")
	dot.Position=UDim2.new(0,1,0,1)
	dot.Size=UDim2.new(0,6,0,6)
	dot.BackgroundColor3=Color3.fromRGB(35,25,55)
	dot.BorderSizePixel=0 dot.ZIndex=12
	dot.Parent=track
	Instance.new("UICorner",dot).CornerRadius=UDim.new(1,0)

	tdots[key]=dot tcolors[key]=color

	local db=false
	track.MouseButton1Click:Connect(function()
		if db then return end db=true
		local on=not tstates[key] tstates[key]=on
		if key=="Kick" then
			if on then core.Mode="Kick" tstates["NoKick"]=false setVisual("NoKick",false)
			else core.Mode="None" end
		elseif key=="NoKick" then
			if on then core.Mode="NoKick" tstates["Kick"]=false setVisual("Kick",false)
			else core.Mode="None" end
		elseif key=="Protector" then core.BorderKick=on
		elseif key=="TpProtector" then core.TpProtector=on
		end
		setVisual(key,on)
		task.delay(0.2,function() db=false end)
	end)
end

makeRow("SPAM STEAL (KICK)",    Color3.fromRGB(255,60,60),  0,"Kick")
makeRow("SPAM STEAL (NO KICK)", Color3.fromRGB(60,255,120), 1,"NoKick")
makeRow("ANTI-TP SCAM",         Color3.fromRGB(255,180,0),  2,"Protector")
makeRow("TP PROTECTOR",         Color3.fromRGB(80,200,255), 3,"TpProtector")

local playerRows={}

local function addPlayerRow(p)
	if p==lp or playerRows[p.UserId] then return end
	local prow=Instance.new("Frame")
	prow.LayoutOrder=1000000+p.UserId%100000
	prow.Size=UDim2.new(1,-2,0,22)
	prow.BackgroundColor3=Color3.fromRGB(4,2,16)
	prow.BackgroundTransparency=0.2
	prow.BorderSizePixel=0 prow.ZIndex=9
	prow.Parent=scroll
	Instance.new("UICorner",prow).CornerRadius=UDim.new(0,4)
	local ps=Instance.new("UIStroke",prow)
	ps.Color=Color3.fromRGB(255,215,0) ps.Thickness=1 ps.Transparency=0.5

	local av=Instance.new("ImageLabel")
	av.Position=UDim2.new(0,2,0.5,-8)
	av.Size=UDim2.new(0,16,0,16)
	av.BackgroundTransparency=1
	av.Image="rbxthumb://type=AvatarHeadShot&id="..p.UserId.."&w=150&h=150"
	av.ZIndex=10 av.Parent=prow
	Instance.new("UICorner",av).CornerRadius=UDim.new(1,0)
	local avs=Instance.new("UIStroke",av) avs.Color=Color3.fromRGB(255,215,0) avs.Thickness=1

	local nl=Instance.new("TextLabel")
	nl.Position=UDim2.new(0,21,0,2)
	nl.Size=UDim2.new(1,-62,0,10)
	nl.BackgroundTransparency=1
	nl.Text=p.Name
	nl.TextColor3=Color3.fromRGB(215,205,255)
	nl.TextSize=6 nl.Font=Enum.Font.GothamBold
	nl.TextXAlignment=Enum.TextXAlignment.Left
	nl.ZIndex=10 nl.Parent=prow

	local rl=Instance.new("TextLabel")
	rl.Position=UDim2.new(0,21,0,12)
	rl.Size=UDim2.new(1,-62,0,8)
	rl.BackgroundTransparency=1
	rl.Text="⭐ player"
	rl.TextColor3=Color3.fromRGB(130,115,185)
	rl.TextSize=5 rl.Font=Enum.Font.Gotham
	rl.TextXAlignment=Enum.TextXAlignment.Left
	rl.ZIndex=10 rl.Parent=prow

	local sb=Instance.new("TextButton")
	sb.Position=UDim2.new(1,-38,0.5,-6)
	sb.Size=UDim2.new(0,35,0,12)
	sb.BackgroundColor3=Color3.fromRGB(60,6,6)
	sb.Text="⚡SPAM"
	sb.TextColor3=Color3.fromRGB(255,215,0)
	sb.TextSize=6 sb.Font=Enum.Font.GothamBold
	sb.AutoButtonColor=false sb.ZIndex=10
	sb.Parent=prow
	Instance.new("UICorner",sb).CornerRadius=UDim.new(0,3)
	local sbs=Instance.new("UIStroke",sb) sbs.Color=Color3.fromRGB(255,215,0) sbs.Thickness=1

	sb.MouseButton1Click:Connect(function()
		if not core.AdminRemote then return end
		fireAdmin("f888ee6e-c86d-46e1-93d7-0639d6635d42",p,"balloon")
		fireAdmin("f888ee6e-c86d-46e1-93d7-0639d6635d42",p,"ragdoll")
		fireAdmin("f888ee6e-c86d-46e1-93d7-0639d6635d42",p,"jumpscare")
		fireAdmin("f888ee6e-c86d-46e1-93d7-0639d6635d42",p,"rocket")
		fireAdmin("f888ee6e-c86d-46e1-93d7-0639d6635d42",p,"jail")
		punishPlayer(p)
	end)

	playerRows[p.UserId]=prow
end

local function removePlayerRow(p)
	if playerRows[p.UserId] then
		playerRows[p.UserId]:Destroy()
		playerRows[p.UserId]=nil
	end
end

for _,p in ipairs(plrs:GetPlayers()) do addPlayerRow(p) end
plrs.PlayerAdded:Connect(addPlayerRow)
plrs.PlayerRemoving:Connect(removePlayerRow)

closebtn.MouseButton1Click:Connect(function() sg.Enabled=false end)

local minimized=false
minbtn.MouseButton1Click:Connect(function()
	minimized=not minimized
	scroll.Visible=not minimized
	footer.Visible=not minimized
	local nh=minimized and 17 or H
	frame.Size=UDim2.new(0,W,0,nh)
	starBG.Size=UDim2.new(0,W,0,nh)
end)

do
	local dragging,dragStart,startPos=false,nil,nil
	tb.InputBegan:Connect(function(i)
		if i.UserInputType==Enum.UserInputType.MouseButton1 or i.UserInputType==Enum.UserInputType.Touch then
			dragging=true dragStart=i.Position startPos=frame.Position
		end
	end)
	tb.InputEnded:Connect(function(i)
		if i.UserInputType==Enum.UserInputType.MouseButton1 or i.UserInputType==Enum.UserInputType.Touch then
			dragging=false
		end
	end)
	uis.InputChanged:Connect(function(i)
		if dragging and (i.UserInputType==Enum.UserInputType.MouseMovement or i.UserInputType==Enum.UserInputType.Touch) then
			local d=i.Position-dragStart
			local np=UDim2.new(startPos.X.Scale,startPos.X.Offset+d.X,startPos.Y.Scale,startPos.Y.Offset+d.Y)
			frame.Position=np
			starBG.Position=np
		end
	end)
end
