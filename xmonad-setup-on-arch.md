# Xmonad

## Installation Addition

```bash
# Nastroje a aplikace, ktere spoustis pres klavesove zkratky v XMonadu:
# - firefox, chromium: webove prohlizece
# - code: Visual Studio Code (open-source verze v Archu)
# - emacs: textovy editor
# - xorg-xset, xorg-xmodmap, xorg-xkill: systemove nastroje pro spravu oken, klavesnice a setreni obrazovky
pacman -S firefox chromium code emacs xorg-xset xorg-xmodmap xorg-xkill

# POZNAMKA PRO PROHLIZEC OBSIDIAN A LOGSEQ:
# Obsidian je v oficialnim repozitari Archu:
pacman -S obsidian

# Logseq a aplikaci Cohesion (predpokladam Flatpak app pro Notion/Notes) 
# bude nejlepsi nainstalovat bud pres Flatpak nebo z AUR (Arch User Repository), 
# protoze nejsou v zakladnich repozitarich. Flatpak pro ne nainstalujes takto:
pacman -S flatpak
flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo
flatpak install flathub com.logseq.Logseq
flatpak install flathub io.github.brunofin.Cohesion

pacman -S ttf-codenewroman-nerd  # Nainstaluje presne ten font, ktery pouzivas (polybar)
```

## Imports & Options

```haskell
{-# OPTIONS_GHC -Wno-deprecations #-}

-- Importy - zakladni funkcionalita xmonad a jeho rozsireni
import XMonad
import XMonad.Layout.ThreeColumns
import XMonad.Layout.Magnifier
import XMonad.Layout.Spacing
import XMonad.Hooks.ManageDocks (avoidStruts, docks, ToggleStruts(..))
import XMonad.Hooks.EwmhDesktops
import XMonad.Hooks.ManageHelpers
import XMonad.Util.EZConfig
import XMonad.Util.Ungrab
import XMonad.Util.SpawnOnce

-- Rychle prepinani workspace (ploch)
import XMonad.Actions.CycleWS
```

## Layouts & ManageHook

```haskell
-- Layouts - definice rozlozeni oken na obrazovce
myLayout = avoidStruts
  (spacingRaw True (Border 5 5 5 5) True (Border 5 5 5 5) True
    (tiled ||| Mirror tiled ||| Full ||| threeCol))
  where
    -- Tri sloupce, kde prostredni sloupec je hlavni (master) a aktivni okno se zvetsi o 30%
    threeCol = magnifiercz' 1.3 $ ThreeColMid nmaster delta ratio
    -- Klasicke rozlozeni: velke okno vlevo, ostatni naskladana vpravo nad sebou
    tiled    = Tall nmaster delta ratio
    -- Vychozi pocet oken v hlavnim (master) panelu
    nmaster  = 1
    -- Vychozi procento obrazovky, ktere zabira hlavni panel (50%)
    ratio    = 1/2
    -- Procento, o kolik se zmeni velikost panelu pri rucnim resizu (3%)
    delta    = 3/100

-- Managehook - specialni pravidla pro konkretni aplikace a typy oken
myManageHook = composeAll
  [ className =? "Gimp" --> doFloat      -- Gimp se vzdy spusti jako plovouci okno
  , isDialog            --> doFloat      -- Jakakoliv dialogova okna (ulozeni souboru, potvrzeni) budou plovouci
  ]
```

## StartupHook & MyConfig

```haskell
-- StartupHook - prikazy a programy, ktere se spusti pri startu XMonadu
myStartupHook :: X ()
myStartupHook = do
    spawnOnce "nitrogen --restore &" -- Obnovi tapetu plochy nastavenou pres program nitrogen
    spawnOnce "picom &"              -- Zapne kompozitor (stiny, pruhlednost oken, plynulost)
    
    -- Zabije pripadny starejsi polybar a spusti novy s tvou konfiguraci
    spawnOnce "pkill polybar; polybar mybar --config='~/.config/polybar/config.ini'"
    
    -- Vypnuti vestaveneho xorg setrice a uspavani monitoru (at ti nezhasina obrazovka)
    spawnOnce "xset s off"
    spawnOnce "xset -dpms"
    spawnOnce "xset s 0 0"
    
    -- Nacte tve vlastni mapovani klaves (pokud pouzivas soubor .Xmodmap v domovskem adresari)
    spawnOnce "xmodmap ~/.Xmodmap"

-- Configuration - hlavni nastaveni XMonadu
myConfig = def
  { modMask            = mod4Mask     -- Nastavi klavesu Windows (Super) jako hlavni Mod klavesu
  , terminal           = "kitty"      -- Vychozi terminal (nainstalovali jsme v prvnim kroku)
  , layoutHook         = myLayout     -- Pouzije nase nadefinovane rozlozeni oken
  , manageHook         = myManageHook -- Pouzije nase specialni pravidla pro okna
  , startupHook        = myStartupHook-- Spusti programy po startu
  , borderWidth        = 4            -- Tloustka ramecku kolem oken v pixelech
  , focusedBorderColor = "#00ffcc"    -- Barva ramecku aktivniho okna (krasna cyan/tyrkysova)
  , normalBorderColor  = "#ebdbb2"    -- Barva ramecku neaktivnich oken (prijemna gruvbox sedo-zluta)
  }
```

## Main & Keybindings

```haskell
-- Main - vstupni bod XMonadu, kde se vsechno spojuje dohromady
main :: IO ()
main =
    xmonad
      $ docks  -- Automaticky uvolni misto pro panely (Polybar/Xmobar) a nenecha okna je prekryvat
      $ ewmh   -- Povoli EWMH podporu (nutne pro spravne zobrazeni ploch v Polybaru a komunikaci s aplikacemi)
      $ myConfig
        `additionalKeysP`
        [ -- Ovladani hlasitosti zvuku (PipeWire/PulseAudio) pres multimedialni klavesy
          ("<XF86AudioRaiseVolume>", spawn "pactl set-sink-volume @DEFAULT_SINK@ +5%")
        , ("<XF86AudioLowerVolume>", spawn "pactl set-sink-volume @DEFAULT_SINK@ -5%")
        , ("<XF86AudioMute>",        spawn "pactl set-sink-mute @DEFAULT_SINK@ toggle")
        
        -- (Zakomentovano) Foceni vyrezu obrazovky nastrojem scrot
        -- , ("M-C-s", unGrab *> spawn "scrot -s")

          -- Rychle prepinani workspace (ploch)
          -- - Linearni pohyb (doleva/doprava) pouze mezi plochami, kde opravdu bezi nejake aplikace
        , ("M-<Right>", moveTo Next NonEmptyWS)
        , ("M-<Left>",  moveTo Prev NonEmptyWS)
          -- - Presune aktualni okno na vedlejsi plochu a hned te tam prepne taky
        , ("M-S-<Right>", shiftToNext >> nextWS)
        , ("M-S-<Left>",  shiftToPrev >> prevWS)
  
          -- Systemove zkratky pro praci s okny a prostredim
        , ("M-b",   sendMessage ToggleStruts) -- Schova/ukaze Polybar, aby mela okna celou plochu
        , ("M-o",   spawn "~/.config/rofi/launchers/type-4/launcher.sh") -- Spusti Rofi menu pro hledani aplikaci
        , ("M-w",   kill) -- Zavre aktualne vybrane okno (klasicke Alt+F4)
        
          -- Spousteni systemovych utilit a souboroveho manazera
        , ("M-.",   spawn "gnome-characters") -- Otevre vyhledavac emoji a znaku
        , ("M-,",   spawn "nautilus") -- Otevre Cinnamon/Gnome file manager
        , ("M-S-'", spawn "gnome-terminal") -- Zalozni terminal (pokud bys potreboval jiny nez Kitty)

          -- Webove prohlizece
        , ("M-f",   spawn "firefox")
        , ("M-S-g", spawn "chromium")

          -- Prace, kod a poznamky
        , ("M-x",   spawn "emacs") -- Textovy editor Emacs
        , ("M-S-l", spawn "logseq") -- Poznamkovy blok Logseq (Flatpak)
        , ("M-S-c", spawn "code") -- VS Code editor
        , ("M-S-o", spawn "gtk-launch obsidian") -- Prohlizec poznamek Obsidian spousteny pres GTK
        , ("M-S-n", spawn "flatpak run io.github.brunofin.Cohesion --disable-spellcheck") -- Notion/Notes aplikace pres Flatpak
        
          -- Nouzove zabiti aplikace (zmeni kurzor na krizek, kliknutim na okno ho natvrdo vypnes)
        , ("C-A-k", spawn "xkill") -- Klávesová zkratka Ctrl + Alt + K
        ]
```

## Keyboard Remapping (.Xmodmap)

```ini
! Caps Lock -> Pravy Ctrl
! Zmeni funkcionalitu klavesy Caps Lock (kod 66) na pravy Control
keycode 66 = Control_R NoSymbol Control_R

! Pravy Ctrl -> Caps Lock
! Zmeni funkcionalitu puvodniho praveho Ctrl (kod 105) na Caps Lock, kdybys ho nekdy nutne potreboval
keycode 105 = Caps_Lock NoSymbol Caps_Lock

! Vyčisteni a znovupriřazeni modifikatoru
! Xorg si musi vnitrne aktualizovat tabulku klaves, aby pochopil, ze Caps Lock uz neni zamek velkych pismen, ale Control
clear Lock
clear Control
add Control = Control_L Control_R
```

# Polybar

## Global Colors and Bar Settings

```sql
[colors]
-- Zakladni tmave gruvbox pozadi a svetle sedy text
background = #1d2021
foreground = #ebdbb2

-- Akcentni barvy (tyrkysova pro ikony, cervena pro ztlumeny zvuk, tmava pro aktivni text)
accent     = #00ccff
muted      = #e16d76
text-dark  = #555555

[bar/mybar]
width  = 100%
height = 29
radius = 2 -- Jemne zaobleni rohu panelu

background = ${colors.background}
foreground = ${colors.foreground}

padding-right = 1

-- Hlavni font pro text i ikony (musi byt nainstalovany Nerd Font, jinak misto ikon uvidis kosticky)
font-0 = "CodeNewRoman Nerd Font:size=16;2"

-- Rozlozeni modulu na liste
modules-left = xworkspaces
modules-center = date
modules-right = cpu sep memory sep pulseaudio tray

-- Nastaveni systemove listy (tray) na prave strane
tray-position = right
tray-padding = 2
tray-scale = 1.0
tray-detached = false
```

## Modules: Workspaces and Separator

```sql
[module/xworkspaces]
type = internal/xworkspaces
pin-workspaces = 1 2 3 4 5 6 7 8 9
enable-click = true  -- Kliknutim mysi na cislo se prepne plocha
enable-scroll = true -- K koleckem mysi nad panelem muzes listovat plochami
enable-urgent = true

-- Aktivni plocha (ta na ktere prave jsi) dostane tyrkysove pozadi a tmavy text
label-active = %name%
label-active-background = ${colors.accent}
label-active-foreground = ${colors.text-dark}
label-active-padding = 1

-- Obsazena plocha (bezi na ni nejaka aplikace, ale nejsi na ni)
label-occupied = %name%
label-occupied-foreground = ${colors.text-dark}
label-occupied-padding = 1

-- Neaktivni (prazdna) plocha
label-inactive = %name%
label-inactive-padding = 1

label-empty = %name%
label-empty-padding = 1

label-urgent = %name%

[module/sep]
-- Jednoduchy textovy oddelovac mezi moduly na prave strane
type = custom/text
content = " | "
```

## Modules: System Resources

```sql
[module/cpu]
type = internal/cpu
interval = 2
format-prefix = "  " -- Ikona procesoru z Nerd Fontu
format-prefix-foreground = ${colors.accent}
format = <label>
label = CPU %percentage:2%%

[module/memory]
type = internal/memory
interval = 2
format-prefix = "  " -- Ikona disku/ramky z Nerd Fontu
format-prefix-foreground = ${colors.accent}
format = <label>
label = RAM %percentage_used:2%%

[module/date]
type = internal/date
interval = 1
date = "%Y/%m/%d %H:%M:%S" -- Format: RRRR/MM/DD HH:MM:SS
label = %date%
format-prefix = "󱑃 " -- Ikona hodin z Nerd Fontu
format-prefix-foreground = ${colors.accent}

[module/pulseaudio]
type = internal/pulseaudio
format-volume-prefix = "  " -- Ikona zvuku z Nerd Fontu
format-volume-prefix-foreground = ${colors.accent}
format-volume = <label-volume>
label-volume = VOL %percentage%%
label-muted = muted
label-muted-foreground = ${colors.muted} -- Pokud je zvuk ztlumeny, zcervena
```