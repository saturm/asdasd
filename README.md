--[[
        Katarina Combo 2.14
                by eXtragoZ
               
        Features:
                - Full combo: DFG -> Q -> E -> W -> Items -> R
                - Auto Ignite in the Full combo (by default off)
                - Supports: Deathfire Grasp, Liandry's Torment, Blackfire Torch, Bilgewater Cutlass, Hextech Gunblade, Blade of the Ruined King, Sheen, Trinity, Lich Bane, Iceborn Gauntlet, Shard of True Ice, Randuin's Omen and Ignite
                - Harass mode:
                        - 0: Q
                        - 1: Q -> W
                        - 2: Q -> E -> W
                - Delay in W to process the mark of Q unless it going to use R
                - Stops the Ult if you can kill your target with basic skills
                - Mark killable target with a combo
                - Target configuration
                - Press shift to configure
   
        Explanation of the marks:
                Green circle:  Marks the current target to which you will do the combo
                Blue circle:  Mark a target that can be killed with a combo, if all the skills were available
                Red circle:  Mark a target that can be killed using items + 1 hits + Q x2 + Qmark x2 + W x2 + E x2 + R (full duration) + ignite
                2 Red circles:  Mark a target that can be killed using items + 1 hits + Q + Qmark + W + E + R (7/10 duration) + ignite
                3 Red circles:  Mark a target that can be killed using items (without on hit items) + Q + Qmark(if e is not on cd) + W(if e is not on cd) + E + R (3/10 duration)(if e is not on cd)
               
        Stops the Ult: Q + Qmark + W + E
]]
if myHero.charName ~= "Katarina" then return end
--[[            Config          ]]    
local HK = 32 --spacebar
local HHK = 84 --T
--[[            Code            ]]
local range = 700 --+65 BB
local ULTK = 82 --R (security method)
local tick = nil
-- Active
local ultActive = false
local timeult = 0
local timeq = 0
local lastqmark = 0
local lastAnimation = ""
-- draw
local waittxt = {}
local calculationenemy = 1
local floattext = {"Skills are not available","Able to fight","Killable","Murder him!"}
local killable = {}
-- ts
local ts
local distancetarget = 0
--
local ID = {DFG = 3128, HXG = 3146, BWC = 3144, Sheen = 3057, Trinity = 3078, LB = 3100, IG = 3025, LT = 3151, BT = 3188, STI = 3092, RO = 3143, BRK = 3153}
local Slot = {Q = _Q, W = _W, E = _E, R = _R, I = nil, DFG = nil, HXG = nil, BWC = nil, Sheen = nil, Trinity = nil, LB = nil, IG = nil, LT = nil, BT = nil, STI = nil, RO = nil, BRK = nil}
local RDY = {Q = false, W = false, E = false, R = false, I = false, DFG = false, HXG = false, BWC = false, STI = false, RO = false, BRK = false}
 
function OnLoad()
        KCConfig = scriptConfig("Katarina Combo 2.14", "katarinacombo")
        KCConfig:addParam("scriptActive", "Combo", SCRIPT_PARAM_ONKEYDOWN, false, HK)
        KCConfig:addParam("harass", "Harass", SCRIPT_PARAM_ONKEYDOWN, false, HHK)
        KCConfig:addParam("harasscombo", "Harass Combo", SCRIPT_PARAM_SLICE, 2, 0, 2, 0)
        KCConfig:addParam("drawcircles", "Draw Circles", SCRIPT_PARAM_ONOFF, true)
        KCConfig:addParam("drawtext", "Draw Text", SCRIPT_PARAM_ONOFF, true)
        KCConfig:addParam("useult", "Use Ult", SCRIPT_PARAM_ONOFF, true)
        KCConfig:addParam("delayw", "Delay W", SCRIPT_PARAM_ONOFF, true)
        KCConfig:addParam("canstopult", "Can stop Ult", SCRIPT_PARAM_ONOFF, true)
        KCConfig:addParam("autoIgnite", "Auto Ignite", SCRIPT_PARAM_ONOFF, false)
        KCConfig:permaShow("scriptActive")
        KCConfig:permaShow("harass")
        ts = TargetSelector(TARGET_LESS_CAST_PRIORITY,range,DAMAGE_MAGIC)
        ts.name = "Katarina"
        KCConfig:addTS(ts)
        if myHero:GetSpellData(SUMMONER_1).name:find("SummonerDot") then Slot.I = SUMMONER_1
        elseif myHero:GetSpellData(SUMMONER_2).name:find("SummonerDot") then Slot.I = SUMMONER_2 end
        for i=1, heroManager.iCount do waittxt[i] = i*3 end
        PrintChat(" >> Katarina Combo 2.14 loaded!")
end
function OnTick()
        ts:update()
        for name,number in pairs(ID) do Slot[name] = GetInventorySlotItem(number) end
        for name,state in pairs(RDY) do RDY[name] = (Slot[name] ~= nil and myHero:CanUseSpell(Slot[name]) == READY) end
        if tick == nil or GetTickCount()-tick >= 100 then
                tick = GetTickCount()
                KCDmgCalculation()
        end
        ultActive = GetTickCount() <= timeult+GetLatency()+50 or lastAnimation == "Spell4"
        if KCConfig.canstopult and ultActive and ts.target ~= nil then
                if KCDmgCalculation2(ts.target) > ts.target.health then ultActive, timeult = false, 0 end
        end    
        if ts.target ~= nil then distancetarget = GetDistance(ts.target) end
        if KCConfig.harass and ts.target ~= nil and not ultActive then 
                if RDY.Q then CastSpell(_Q, ts.target) end
                if KCConfig.harasscombo == 2 and RDY.E then CastSpell(_E,ts.target) end
                if KCConfig.harasscombo >= 1 then
                        if RDY.W and distancetarget<375 and (((GetTickCount()-timeq>650 or GetTickCount()-lastqmark<650) and not RDY.Q) or not KCConfig.delayw) then CastSpell(_W) end
                end
        end
        if KCConfig.scriptActive and ts.target ~= nil and KCConfig.autoIgnite and RDY.I then
                local QWEDmg = KCDmgCalculation2(ts.target)
                local RDmg = (RDY.R and KCConfig.useult and distancetarget<=325) and getDmg("R",ts.target,myHero)*5 or 0
                local IDmg = getDmg("IGNITE",ts.target,myHero)
                if distancetarget<=600 and ts.target.health > QWEDmg+RDmg and ts.target.health <= IDmg+QWEDmg+RDmg then CastSpell(Slot.I, ts.target) end
        end
        if KCConfig.scriptActive and ts.target ~= nil and not ultActive then
                if RDY.DFG then CastSpell(Slot.DFG, ts.target) end
                if RDY.Q then CastSpell(_Q, ts.target) end
                if RDY.E then CastSpell(_E,ts.target) end
                if RDY.W and distancetarget<375 and (((GetTickCount()-timeq>650 or GetTickCount()-lastqmark<650) and not RDY.Q) or not KCConfig.delayw or (KCConfig.useult and RDY.R)) then CastSpell(_W) end
                if RDY.HXG then CastSpell(Slot.HXG, ts.target) end
                if RDY.BWC then CastSpell(Slot.BWC, ts.target) end
                if RDY.BRK then CastSpell(Slot.BRK, ts.target) end
                if RDY.STI and distancetarget<=380 then CastSpell(Slot.STI, myHero) end
                if RDY.RO and distancetarget<=500 then CastSpell(Slot.RO) end
                if RDY.R and KCConfig.useult and not RDY.Q and not RDY.W and not RDY.E and not RDY.DFG and not RDY.HXG and not RDY.BWC and not RDY.BRK and not RDY.STI and not RDY.RO and distancetarget<275 then
                        timeult = GetTickCount()
                        CastSpell(_R)
                end
        end
end
function KCDmgCalculation2(enemy)
        local distanceenemy = GetDistance(enemy)
        local qdamage = getDmg("Q",enemy,myHero)
        local qdamage2 = getDmg("Q",enemy,myHero,2)
        local wdamage = getDmg("W",enemy,myHero)
        local edamage = getDmg("E",enemy,myHero)
        local combo5 = 0
        if RDY.Q then
                combo5 = combo5 + qdamage
                if RDY.E or (distanceenemy<375 and RDY.W) then
                        combo5 = combo5 + qdamage2
                end
        end
        if RDY.W and (RDY.E or distanceenemy<375) then
                combo5 = combo5 + wdamage
        end
        if RDY.E then
                combo5 = combo5 + edamage
        end
        return combo5
end
function KCDmgCalculation()
        local enemy = heroManager:GetHero(calculationenemy)
        if ValidTarget(enemy) then
                local qdamage = getDmg("Q",enemy,myHero)
                local qdamage2 = getDmg("Q",enemy,myHero,2)
                local wdamage = getDmg("W",enemy,myHero)
                local edamage = getDmg("E",enemy,myHero)
                local rdamage = getDmg("R",enemy,myHero) --xdagger (champion can be hit by a maximum of 10 daggers (2 sec))
                local hitdamage = getDmg("AD",enemy,myHero)
                local dfgdamage = (Slot.DFG and getDmg("DFG",enemy,myHero) or 0)--amplifies all magic damage they take by 20%
                local hxgdamage = (Slot.HXG and getDmg("HXG",enemy,myHero) or 0)
                local bwcdamage = (Slot.BWC and getDmg("BWC",enemy,myHero) or 0)
                local brkdamage = (Slot.BRK and getDmg("RUINEDKING",enemy,myHero,2) or 0)
                local ignitedamage = (Slot.I and getDmg("IGNITE",enemy,myHero) or 0)
                local onhitdmg = (Slot.Sheen and getDmg("SHEEN",enemy,myHero) or 0)+(Slot.Trinity and getDmg("TRINITY",enemy,myHero) or 0)+(Slot.LB and getDmg("LICHBANE",enemy,myHero) or 0)+(Slot.IG and getDmg("ICEBORN",enemy,myHero) or 0)
                local onspelldamage = (Slot.LT and getDmg("LIANDRYS",enemy,myHero) or 0)+(Slot.BT and getDmg("BLACKFIRE",enemy,myHero) or 0)
                local onspelldamage2 = 0
                local combo1 = hitdamage + (qdamage*2 + qdamage2*2 + wdamage*2 + edamage*2 + rdamage*10)*(RDY.DFG and 1.2 or 1) + onhitdmg + onspelldamage*4 --0 cd
                local combo2 = hitdamage + onhitdmg
                local combo3 = hitdamage + onhitdmg
                local combo4 = 0
                if RDY.Q then
                        combo2 = combo2 + (qdamage + qdamage2)*(RDY.DFG and 2.2 or 2)
                        combo3 = combo3 + (qdamage + qdamage2)*(RDY.DFG and 1.2 or 1)
                        combo4 = combo4 + qdamage + (RDY.E and qdamage2 or 0)
                        onspelldamage2 = onspelldamage2+1
                end
                if RDY.W then
                        combo2 = combo2 + wdamage*(RDY.DFG and 2.2 or 2)
                        combo3 = combo3 + wdamage*(RDY.DFG and 1.2 or 1)
                        combo4 = combo4 + (RDY.E and wdamage or 0)
                        onspelldamage2 = onspelldamage2+1
                end
                if RDY.E then
                        combo2 = combo2 + edamage*(RDY.DFG and 2.2 or 2)
                        combo3 = combo3 + edamage*(RDY.DFG and 1.2 or 1)
                        combo4 = combo4 + edamage
                        onspelldamage2 = onspelldamage2+1
                end
                if myHero:CanUseSpell(_R) ~= COOLDOWN and not myHero.dead then
                        combo2 = combo2 + rdamage*10*(RDY.DFG and 1.2 or 1)
                        combo3 = combo3 + rdamage*7*(RDY.DFG and 1.2 or 1)
                        combo4 = combo4 + (RDY.E and rdamage*3 or 0)
                        onspelldamage2 = onspelldamage2+1
                end
                if RDY.DFG then
                        combo1 = combo1 + dfgdamage
                        combo2 = combo2 + dfgdamage
                        combo3 = combo3 + dfgdamage
                        combo4 = combo4 + dfgdamage
                end
                if RDY.HXG then              
                        combo1 = combo1 + hxgdamage*(RDY.DFG and 1.2 or 1)
                        combo2 = combo2 + hxgdamage*(RDY.DFG and 1.2 or 1)
                        combo3 = combo3 + hxgdamage*(RDY.DFG and 1.2 or 1)
                        combo4 = combo4 + hxgdamage
                end
                if RDY.BWC then
                        combo1 = combo1 + bwcdamage*(RDY.DFG and 1.2 or 1)
                        combo2 = combo2 + bwcdamage*(RDY.DFG and 1.2 or 1)
                        combo3 = combo3 + bwcdamage*(RDY.DFG and 1.2 or 1)
                        combo4 = combo4 + bwcdamage
                end
                if RDY.BRK then
                        combo1 = combo1 + brkdamage
                        combo2 = combo2 + brkdamage
                        combo3 = combo3 + brkdamage
                        combo4 = combo4 + brkdamage
                end
                if RDY.I then
                        combo1 = combo1 + ignitedamage
                        combo2 = combo2 + ignitedamage
                        combo3 = combo3 + ignitedamage
                end
                combo2 = combo2 + onspelldamage*onspelldamage2
                combo3 = combo3 + onspelldamage/2 + onspelldamage*onspelldamage2/2
                combo4 = combo4 + onspelldamage
                if combo4 >= enemy.health then killable[calculationenemy] = 4
                elseif combo3 >= enemy.health then killable[calculationenemy] = 3
                elseif combo2 >= enemy.health then killable[calculationenemy] = 2
                elseif combo1 >= enemy.health then killable[calculationenemy] = 1
                else killable[calculationenemy] = 0 end  
        end
        if calculationenemy == 1 then calculationenemy = heroManager.iCount
        else calculationenemy = calculationenemy-1 end
end
function OnProcessSpell(unit, spell)
        if unit.isMe and spell.name == "KatarinaQ" then timeq = GetTickCount() end
end
function OnCreateObj(object)
        if object.name:find("katarina_daggered") then lastqmark = GetTickCount() end
end
function OnAnimation(unit,animationName)
        if unit.isMe and lastAnimation ~= animationName then lastAnimation = animationName end
end
function OnDraw()
        if KCConfig.drawcircles and not myHero.dead then
                DrawCircle3D(myHero.x, myHero.y, myHero.z, range, 1, ARGB(255,25,255,18))
                if ts.target ~= nil then
                        DrawCircle3D(ts.target.x, ts.target.y, ts.target.z, ts.target.boundingRadius, 8, ARGB(200,255,155,0), 20)
                end
        end
        for i=1, heroManager.iCount do
                local enemydraw = heroManager:GetHero(i)
                if ValidTarget(enemydraw) then
                        if KCConfig.drawcircles then
                                if killable[i] == 1 then
                                        DrawCircle3D(enemydraw.x, enemydraw.y, enemydraw.z, 30, 6, ARGB(155,0,0,255), 16)
                                elseif killable[i] == 2 then
                                        DrawCircle3D(enemydraw.x, enemydraw.y, enemydraw.z, 10, 6, ARGB(155,255,0,0), 16)
                                elseif killable[i] == 3 then
                                        DrawCircle3D(enemydraw.x, enemydraw.y, enemydraw.z, 10, 6, ARGB(155,255,0,0), 16)
                                        DrawCircle3D(enemydraw.x, enemydraw.y, enemydraw.z, 30, 6, ARGB(155,255,0,0), 16)
                                elseif killable[i] == 4 then
                                        DrawCircle3D(enemydraw.x, enemydraw.y, enemydraw.z, 10, 6, ARGB(155,255,0,0), 16)
                                        DrawCircle3D(enemydraw.x, enemydraw.y, enemydraw.z, 30, 6, ARGB(155,255,0,0), 16)
                                        DrawCircle3D(enemydraw.x, enemydraw.y, enemydraw.z, 50, 6, ARGB(155,255,0,0), 16)
                                end
                        end
                        if KCConfig.drawtext and waittxt[i] == 1 and killable[i] ~= 0 then
                                PrintFloatText(enemydraw,0,floattext[killable[i]])
                        end
                end
                if waittxt[i] == 1 then waittxt[i] = 30
                else waittxt[i] = waittxt[i]-1 end
        end
end
function OnWndMsg(msg,key)
        if key == ULTK and msg == KEY_DOWN then timeult = GetTickCount() end
end
