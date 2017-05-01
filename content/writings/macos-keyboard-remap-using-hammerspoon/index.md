---
title: MacOS keyboard remap using Hammersppon
classes: long-form
---
February 2017  
Originally published on [infi.nl](https://infi.nl/nieuws/macos-keyboard-remappen-met-hammerspoon/) (Dutch)

---
Hoewel ik geen VIM-adept ben zoals sommige van mijn collega’s hou ik graag mijn handen op het toetsenbord als ik aan het typen ben. Het liefst hou ik ze ook nog eens op dezelfde plek.

Nu komt het nogal regelmatig voor dat je de cursor wil verplaatsen of een menu-item moet selecteren en, als je niet met de muis wil werken, heb je daar de pijltjestoetsen voor. Daar komt echter het ‘op dezelfde plek’ om de hoek kijken: je moet je handen verplaatsen.  Aangezien ik liever lui dan moe ben zocht ik daar een oplossing voor.

In VIM heb je bijvoorbeeld home row arrows: je gebruikt de toetsen `hjkl` om je cursor te verplaatsen (om tekst in te voeren moet je eerst switchen naar *insert mode*).  Dat is in elke andere editor natuurlijk niet zo handig omdat, als je daar die lettertoetsen gebruikt, je gewoon `hjkl` typt.

Daarnaast vind ik persoonlijk `hjkl` niet zo relaxed (al zal dat wel gewenning zijn), en besloot ik te zoeken naar een manier om `esdf` te gebruiken, wat ook mijn voorkeurstoetsen zijn om in games rond te bewegen. En ik wil het natuurlijk overal kunnen gebruiken, en niet alleen maar in één programma.

## De eerste remapstap
Een aantal jaar geleden ontdekte ik KeyRemap4MacBook, wat tegenwoordig door het leven gaat als [Karabiner](https://pqrs.org/osx/karabiner/). Daarmee kan je, zoals de oude naam al aangeeft, toetsen (en toetscombinaties) remappen naar andere. Het is niet het meest gebruiksvriendelijke programma, maar dat mag de pret niet drukken.

Het komt met een lading standaardregels, en je kan ook zelf regels opstellen zoals “remap `a` naar `e` en `e` naar `a`” (els ja det hendig zou vindan), of “remap `⌃L` naar `⌘⌥⎋`” waardoor je het force-quit-scherm krijgt als je op `⌃L` drukt, en nog veel ingewikkeldere dingen.

Karabiner kan ook onderscheid maken tussen linker- of rechter modifiers (shift (`⇧`), control (`⌃`), alt (`⌥`) en command (`⌘`) zijn modifiers) en daar zag ik mijn oplossing: wat als ik nou de *rechter* command-toets gebruik om `esdf` te remappen naar de pijltjestoetsen? Zo gezegd, zo gedaan en sinds die tijd gebruik ik die combinatie.

Mijn rechterduim iets naar rechts en ik heb mijn pijltjestoetsen onder mijn linkerhand. Als ik shortcuts nodig heb die de een van de combinaties `⌘` + `esdf` nodig heeft (zoals `⌘S` om een bestand op te slaan) kan ik die gebruiken met de linkercommandtoets en ik mis geen functionaliteit.

## De zoektocht naar vervanging...
Na jaren van gemak kwam daar een paar maanden geleden macOS Sierra uit, met een heleboel verbeteringen – maar ook een verslechtering: er was het één en ander veranderd aan de keyboardhandling en ineens werkte Karabiner niet meer. Plotseling was ik weer veroordeeld tot de pijltjestoetsen!

De ontwikkelaar van Karabiner besloot dat het daarom tijd was om Karabiner te laten vallen en een nieuwe, gebruiksvriendelijkere applicatie in het leven te roepen: [Karabiner-Elements](https://github.com/tekezo/Karabiner-Elements). Hij werkt er hard aan, maar tot op heden is er nog geen ondersteuning om toets*combinaties* te remappen, enkel losse toetsen.

Dat is leuk als je de `a` en `e` wilt verwisselen en natuurlijk ook andere nuttigere dingen, maar voor mijn scenario heb je daar dus helemaal niets aan.

## Hammerspoon
In de issuetracker van Elements wordt druk gediscussieerd en daar werd [Hammerspoon](http://www.hammerspoon.org/) getipt: een applicatie waarmee je ontzettend veel dingen in macOS kan automatiseren door middel van [Lua-script](http://www.lua.org/about.html). Ik ben niet bepaald een groot fan van Lua, maar besloot toch maar eens in Hammerspoon te duiken, want misschien kon dit mijn problemen oplossen.

Hammerspoon biedt standaard [hotkey-functionaliteit](http://www.hammerspoon.org/docs/hs.hotkey.html) aan, en dat leek op het eerste gezicht wel te doen wat ik wilde. Om bijvoorbeeld `⌘S` te gebruiken als `←` zou je de volgende code kunnen gebruiken:

```lua
hs.hotkey.bind({"command"}, "s", function()
  hs.eventtap.keyStroke({""}, "left")
end, nil, function()
  hs.eventtap.keyStroke({""}, "left")
end)
```

Echter, op deze manier kan je geen onderscheid maken tussen de linker of rechter commandtoets dus loop je er al snel tegenaan dat je het script wat je aan het typen bent niet eens meer op kan slaan met `⌘S`.

Daarna heb ik nog heel even overwogen dan maar andere modifiers te gebruiken, bijvoorbeeld `⌘⌥`, maar daar was ik ook snel van genezen: behalve dat ik mijn rechterduim in een onmogelijke hoek moet vouwen worden de toetsen zoals de functienaam al aangeeft als *hotkey* geregistreerd en dat betekent dat het ook interfereert met andere hotkeys. Dus `⌘⌥D`, wat voor mij de cursor om laat moet bewegen, togglet ook standaard de zichbaarheid van je dock. Ik moest een andere oplossing vinden.

Hammerspoon proxiet heel veel dingen van macOS direct door, en kan zodoende ook een [eventtap](http://www.hammerspoon.org/docs/hs.eventtap.html) plaatsen op de keyboardevents en deze events aanpassen. De standaard-functionaliteit die dat soort events bieden is wat beperkt (zo kan je bijvoorbeeld niet checken of een linker- of rechter modifier wordt gebruikt) maar gelukkig is de originele eventdata ook te raadplegen en daar staat die informatie wél in.

## Aan de slag
Laten we beginnen met het plaatsen van een tap op de `keyDown` en `keyUp` events, want die willen we aanpassen (key repeats zijn herhalingen van `keyDown`, voorzien een extra flag die ons nu verder niet interesseert):

```lua
local tap ; tap = hs.eventtap.new({
  hs.eventtap.event.types.keyDown,
  hs.eventtap.event.types.keyUp
}, function(event)
  -- onze code komt hier
  return false
end):start()
```

De functie returnt `false`; als we `true` zouden returnen zou Hammerspoon het event weggooien en dat willen we niet: we willen het aanpassen en daarna zijn weg door de rest van de software laten vervolgen.

We halen eerst de `flags` op uit het originele event, want daar moeten we de informatie over de modifiers uithalen. Dit is een integer waarin bepaalde bits zijn aan- of uitgezet afhankelijk van welke modifiers worden gebruikt. Zo staat bit `21` op `1` als er een command-toets is ingedrukt. Bits `4` en `5` geven vervolgens aan of het de linker of rechter is (of beide).

```lua
local flags = event:getRawEventData().CGEventData.flags

local cmd = flags & 0x100000 > 0
local cmdLeft = cmd and flags & 0x8 > 0
local cmdRight = cmd and flags & 0x10 > 0
view rawhammerspoon3 hosted with ❤ by GitHub
Daarna moeten we checken of we een remap willen doen:

local keyCode = event:getKeyCode()

-- if right command && (e, s, d or f)
if (cmdRight and (keyCode == 14 or keyCode == 1 or keyCode == 2 or keyCode == 3)) then
  -- remap!
end
```

Oké, nu is de rechtercommand gebruikt met `esd` of `f`. Dan wordt het nu een beetje lastig: we willen namelijk de *rechter* command uit het event weglaten maar de overige modifiers behouden, zodat we bijvoorbeeld `⌘⌥S` kunnen gebruiken als `⌥←` om een woord naar links te springen, maar als ook de linkercommand wordt gebruikt willen we de command-modifier behouden zodat `⌘S` (met *beide* command-toetsen!) vertaald wordt naar `⌘←` om naar het begin van een regel te gaan). Daarnaast kunnen we de `flags` niet aanpassen: die zijn read-only in deze interface.

We kunnen wel iets minder nauwkeurige `flags` van het event verkrijgen, dat is een `table` (de meeste andere talen noemen dat een dictionary) met daarin welke modifiers zijn gebruikt. Als we daarin dan cmd op false zetten en hem weer terugvoeren is de command-modifier uitgeschakeld.

Hiermee zijn we echter ook de informatie kwijt of er (bijvoorbeeld) de linker of rechter alt is gebruikt: enkel de informatie dat er een `⌥` is gebruikt blijft behouden. Ik heb er zelf geen last van, maar het kan iets zijn waar je voor jouw configuratie rekening mee moet houden.

```lua
if (not cmdLeft) then
  local flags = event:getFlags()
  flags["cmd"] = false
  event:setFlags(flags)
end
```

Vervolgens is het alleen nog maar een kwestie van het aanpassen van de keycode:

```lua
-- remap esdf to arrows
if (keyCode == 14) then event:setKeyCode(126) end -- up
if (keyCode == 1) then event:setKeyCode(123) end -- left
if (keyCode == 2) then event:setKeyCode(125) end -- down
if (keyCode == 3) then event:setKeyCode(124) end -- right
```

Op zich zijn we nu klaar met het remappen van de toetsen. Helaas kwam ik nog een bug tegen: in sommige gevallen vindt er een time-out plaats en wordt de eventtap uitgeschakeld.

Gelukkig geeft de discussie bij die bugmelding ook een oplossing: er wordt namelijk een event gestuurd met de melding dat de tap wordt uitgeschakeld en op dat moment kunnen we hem direct weer inschakelen.

Dat betekent wel dat we ook naar die “uitschakel-events” moeten gaan luisteren:

```lua
local tap ; tap = hs.eventtap.new({
  hs.eventtap.event.types.keyDown,
  hs.eventtap.event.types.keyUp,
  hs.eventtap.event.types.tapdisabledbytimeout,
  hs.eventtap.event.types.tapdisabledbyuserinput
}, function(event)
  -- remap code
end
```

En daar moeten we natuurlijk ook iets mee doen, namelijk als die optreedt moeten we de tap opnieuw starten (daara kunnen we gelijk returnen, natuurlijk).

```lua
local type = event:getType()
if (type == hs.eventtap.event.types.tapdisabledbytimeout or
    type == hs.eventtap.event.types.tapdisabledbyuserinput) then
  tap:start()
  return true
end
```

Ik moet je wel heel eerlijk bekennen dat ik deze oplossing nog niet echt getest heb: sinds ik deze fix heb geïntroduceerd heb ik geen time-out meer gehad, dus ik heb geen idee of het ook echt werkt.
Het complete scriptje ziet er nu zo uit:

```lua
local tap ; tap = hs.eventtap.new({
  hs.eventtap.event.types.keyDown,
  hs.eventtap.event.types.keyUp,
  hs.eventtap.event.types.tapdisabledbytimeout,
  hs.eventtap.event.types.tapdisabledbyuserinput
}, function(event)

  local type = event:getType()
  if (type == hs.eventtap.event.types.tapdisabledbytimeout or
      type == hs.eventtap.event.types.tapdisabledbyuserinput) then
    tap:start()
    return true
  end

  local flags = event:getRawEventData().CGEventData.flags

  local cmd = flags & 0x100000 > 0
  local cmdLeft = cmd and flags & 0x8 > 0
  local cmdRight = cmd and flags & 0x10 > 0

  local keyCode = event:getKeyCode()

  -- if right command && (e, s, d or f)
  if (cmdRight and (keyCode == 14 or keyCode == 1 or keyCode == 2 or keyCode == 3)) then

    if (not cmdLeft) then
      local flags = event:getFlags()
      flags["cmd"] = false
      event:setFlags(flags)
    end

    -- remap esdf to arrows
    if (keyCode == 14) then event:setKeyCode(126) end -- up
    if (keyCode == 1) then event:setKeyCode(123) end -- left
    if (keyCode == 2) then event:setKeyCode(125) end -- down
    if (keyCode == 3) then event:setKeyCode(124) end -- right

  end

  return false
end):start()
```

Hoewel dit een hoop uitzoekwerk was heeft het wel mijn interesse in Hammerspoon gewekt want dit is slechts een heel klein deel van de mogelijkheden die het biedt. Kijkt maar eens op de [Getting started-pagina](http://www.hammerspoon.org/go/), en daarna bij de voorbeeldconfiguraties. Daar zitten hele coole dingen tussen, en wellicht ook de functionaliteit die je al jaren mist.
