# retrospective explanation
I've reached the conclusion of my work on this issue and don't want to submit it upstream at this time. Here is the explanation:

- Kakoune requests kitty keyboard protocol levels 1 and 5
  - KKP level 1 means that keys with modifiers are encoded with the modifiers, and notably if shift is used it sends shift as one of the modifiers and the unshifted key as the main key
    - for example, alt+shift+s is sent (with a specific encoding) as alt+shift+s instead of alt+S (alt and capital s)
  - KKP level 5 means that an "alternate key" is sent with certain key presses, so alt+shift+s is sent as alt+shift+s with the alternate key being `S` (capital s)
  - Kakoune relies on KKP level 5 to receive the "alternate key". If a key it receives (with KKP) has the shift modifier, it checks if there is an alternate key supplied, and if so, it drops the shift modifier and sets the main key of the key press to that alternate key
    - So if it receives alt+shift+s with alt key `S`, which would be the normal thing that an emulator that supports KKP would send for the keypress "alt+shift+s", kakoune turns it into alt+S
  - When creating key mappings in kakoune, the mapping is "canonicalized", which includes dropping shift and uppercasing the key if it is a letter, or throwing an error if it isn't a letter
    - ex. `<a-s-w>`, which is alt+shift+w, is canonicalized to `<a-W>`
    - ex. `<a-s-0>`, alt+shift+0, throws an error because kakoune doesn't handle turning shift+0 into something else, it believes that should be the terminal emulator's job
    - see `canonicalize_ifn` in `src/keys.cc`)
- Kakoune does not check if the terminal emulator actually supports certain KKP levels. It's input parsing assumes that if it's getting KKP encoded presses, the emulator supports level 1 and 5
- Zellij supports only kitty keyboard protocol (KKP) level 1, see [here](https://github.com/zellij-org/zellij/issues/3789) for some justification
  - This means that kakoune gets alt+shift+w from zellij, but since there's no alternate key, kakoune treats it as `<a-s-w>`, which there cannot be a mapping for, since it is canonicalized to `<a-W>`
- This "fix" simply "canonicalizes" keypresses as they are coming in by dropping shift and uppercasing the key if it is a letter. This means that `<a-s-w>` in zellij is properly turned into `<a-W>`.
  - the fix doesn't work for non-letter keys, because kakoune does not have any logic for knowing what the shifted version of non-letter keys should be
- I've decided not to upstream this because it is only a partial fix, and I consider it to be a problem that zellij should solve. Zellij's maintainer believes that it is perfectly valid to only support KKP level 1, and the KKP does make provisions for this, however it means that every terminal application that runs in zellij needs to know what the shifted versions of keys are. This is silly, and should be handled at the emulator level, which KKP recognizes. The KKP progressive enhancements are intended more for terminal applications than emulators. While zellij is sort of both, it should be considered more as a terminal emulator than an application.
  - Ultimately this is just another frustration and poor choice that I see made by Zellij, and is probably the straw that breaks the camel's back and gets me to drop zellij 

# shift ctrl while in zellij
I've created a binding to shift-ctrl-w:

```
map global normal '<s-c-w>' 'ia' 
```

It works when I run kakoune simply inside kitty, but not when I run kakoune inside zellij (or tmux) inside kitty (or other terminal emulators I've tried). The goal is to find out why and fix it.

# things I've tried
- map `<c-W>` instead of `<s-c-w>`, same behavior 
- map `<c-s-w>` instead of `<s-c-w>`, same behavior
- setting in `~/.config/zellij/config.kdl` both `support_kitty_keyboard_protocol false` and `true`
  - most testing done with `support_kitty_keyboard_protocol false`

# information
## showkey output
- showkey outputs `<ESC>[119;6u` in both just kitty and zellij (with `support_kitty_keyboard_protocol false`) in kitty
  - this is good and expected, `https://www.leonerd.org.uk/hacks/fixterms/` specifies the "fixterms" protocol that allows extended keys. `<ESC>[119;6u` is the escape sequence, followed by char code 119 (lowercase w), then `;6` indicates the modifier keys, which is shift(1)+ctrl(1)+offset(1), see the protocol link to verify

## kakoune debug key output
`:set global debug keys` in kak logs keys received to the debug buffer.
- in just kitty: `got key '<c-W>'`
- in zellij in kitty: `got key '<c-s-w>'`

## files
- `src/keys.cc`: maybe just for parsing of key strings? seems to output debug output for some key presses, but not what expecting
  - `canonicalize_ifn`
  - :

## kitty keyboard protocol vs fixterms

## discovered so far
- kakoune "canonicalizes" key mappings, and this includes removing the shift modifier and uppercasing the character, ex. `<c-s-w>` becomes `<c-W>`
  - see `canonicalize_ifn` in `src/keys.cc`
- canonicalization does not happen when reading keys from the terminal
  - key input handling is in `get_next_key()` in `src/terminal_ui.cc`
    - see `masked_key`
- kakoune requests kitty keyboard protocol "disambiguate escape codes" and "report alternate keys" progressive enhancements by sending `\033[>5u`
  - see `setup_terminal()` in `src/terminal_ui.cc`
- zellij only supports the "disambiguate escape codes" enhancement, not the "report alternate keys" enhancement
  - see https://github.com/zellij-org/zellij/issues/3789
  - kakoune does not check what level of the kitty keyboard protocol is supported (or maybe whether it is supported at all?)
- for ctrl-shift-w, in zellij kakoune receives `<ESC>[119;6u`, which is lowercase w with the ctrl and shift modifiers
  - in plain kitty kakoune receives `<ESC>[119:87;6u`, which is lowercase w with uppercase W as an alternate key, with the ctrl and shift modifiers pressed

## possible solutions
- add uppercasing and shift-dropping in `masked_key` (`src/terminal_ui.cc:831`)
  - implemented, working

## tmux testing
tmux has support for the "fixterms" spec. See here:
- https://github.com/tmux/tmux/wiki/Modifier-Keys

I am trying to test how my implementation of "canonicalizing" letters with the "shift" modifier works with fixterms in tmux.

Testing in iterm2, with CSI U mode enabled
- in plain iterm, after starting and running `showkey`, ctrl+shift+w outputs `<ESC>[87;5u`, which is what I expect
  - if I first run `printf '\033[>4;1m'`, which I thought signaled to turn fixterms mode on, it then showkey outputs `<CTL-W=ETB>` (i.e. plain ctrl+w)

# todo
- [x] verify that [[#showkey output]] is correct according to the kitty keyboard protocl
- [x] find out why shift+alt+w works but shift+ctrl+w doesn't
  - solution: when zellij has `support_kitty_keyboard_protocol true`, then alt+shift+w is reported as a-s-w, when it is `false`, then ctrl+shift uses the CSI-U format because `setup_terminal` sends `\033[>4;1m` to request fixterms-style CSI-U key reporting, which does not use CSI U for alt+shift. So it's only the case that shift+alt works while ctrl+alt doesn't when KKP is not supported at all. 
  - showkey output for shift+alt+w:
    - zellij in kitty with `\033[>4;1m` (CSI u) sent followed by `\033[>5u` (kitty keyboard protocol): `<ESC>[119;4u`
    - just kitty with `\033[>4;1m` (CSI u) sent followed by `\033[>5u` (kitty keyboard protocol): `<ESC>[119:87;4u`
  - [x] create a debug statement to see exactly what bytes are being received
    - shows that in zellij, alt+shift+w is using traditional encoding, not CSI-U. CSI-U is used in plain kitty

# llm prompts
## why does alt shift work but ctrl shift doesn't
I'm trying to understand why some CSI-U sequences are working and some are not. Here is what I've gathered:

- zellij only supports level 1 of the kitty keyboard protocol (disambiguate escape codes). This means that it sends CSI-U sequences, but it does *not* send the "shifted key"
- kakoune canonicalizes key mappings, so `<c-s-w>` is canonicalized to `<c-W>` and `<a-s-w>` is canonicalized to `<a-W>`
- When I run kakoune in plain kitty (not in zellij), which supports the full kitty keyboard protocol, ctrl+shift+w sends `<ESC>[119:87;6u` and alt+shift+w sends `<ESC>[119:87;4u`. 119 is lowercase w, and 87 is uppercase W as the "shifted key". Kakoune looks for this shifted key (see `src/terminal_ui.cc:829`, `masked_key`), and if it is present, it drops the shift modifier and replaces the key with the shifted key.
  - If I run `set global debug keys` in kakoune, I can see that it reports `got key '<c-W>'` or `got key '<a-W>'`
- When I run kakoune in zellij, since zellij does not support the "report alternate keys" feature, ctrl+shift+w sends `<ESC>[119;6u` and alt+shift+w sends `<ESC>[119;4u`. Here is what I don't understand: since kakoune does not receive a "shifted key" parameter in the escape code sent by zellij, kakoune treats ctrl-shift-w (which sends `<ESC>[119;6u`) as `got key '<c-s-w>'`. This makes sense based on `masked_key`. However, alt+shift+w (which sends `<ESC>[119;4u`) still works, and kakoune recognizes it as `got key '<a-W>'`. How? Why is it correctly parsing alt+shift in this case, but not ctrl+shift?

# resources
- https://github.com/zellij-org/zellij/issues/3789
  - https://github.com/zellij-org/zellij/issues/3897
- https://github.com/mawww/kakoune/issues/5271
