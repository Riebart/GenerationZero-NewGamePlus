# GenerationZero-NewGamePlus
Provide a New Game Plus function that using direct save game file modifications.

## Proof of Concept

To test this out:

- Download the deca GUI from [the releases](https://github.com/kk49/deca/releases).
- Load your savegame files from `C:\Users\${Username}\Documents\Avalanche Studios\GenerationZero\Saves\${SteamId}\savegame` using `File -> Add External`
- Check the "Export as Text" checkbox, then expand the tree in the top left tree GUI box to show your loaded save game file.
- Click the Extract button (two to the right of  "Export Raw Files"), wait for the CPU usage to die down.
- Here's some bash that'll eat that text output, and then overwrite the relevant parts of your savegame.

```bash
cat savegame.txt | sed -n '/CompletionState:/{N;s/\n//;s/\r//;p}' | tr -s ' ' | grep "CompletionState:" | sed 's/^.*Data Offset: \([0-9]*\)[^0-9].*$/\1/' | while read offset; do dd if=/dev/zero of=savegame.reset bs=1 count=1 conv=notrunc seek=$offset 2>/dev/null >/dev/null; echo "$offset"; done | pv -l -s 700 > /dev/null
```
  
  - The basic idea is to reset every `CompletionState` value to 0. This is kind of what a fresh start looks like.
