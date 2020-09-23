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
cat savegame.txt | \
  sed -n '/CompletionState:/{N;s/\n//;s/\r//;p}' | \
  tr -s ' ' | grep "CompletionState:" | \
  sed 's/^.*Data Offset: \([0-9]*\)[^0-9].*$/\1/' | \
  while read offset
  do
    dd if=/dev/zero of=savegame.reset bs=1 count=1 conv=notrunc seek=$offset 2>/dev/null >/dev/null
    echo "$offset"
  done | pv -l -s 700 > /dev/null
```
  
  - The basic idea is to reset every `CompletionState` value to 0. This is kind of what a fresh start looks like.

## Building on Ubuntu 20

### Setting up `deca`

```bash
apt update
DEBIAN_FRONTEND=noninteractive apt install -y build-essential git python3-pip python3-numpy python3-matplotlib pv
git clone https://github.com/kk49/deca.git
cd deca
pip3 install -r <(cat requirements.txt | cut -d '=' -f1)
pip3 install zugbruecke
```

### An updated `process_adf.py`

```python
import sys
from deca.file import ArchiveFile
from deca.ff_adf import Adf


class FakeVfs:
    def hash_string_match(self, hash32=None, hash48=None, hash64=None):
        return []
    def lookup_equipment_from_hash(self, hash):
        return None

in_file = sys.argv[1]

obj = Adf()
with ArchiveFile(open(in_file, 'rb')) as f:
    obj.deserialize(f)

print(obj.dump_to_string(FakeVfs()))
```

### Resetting mission progress

```bash
time python3 process_adf.py ~/savegame > ~/savegame.txt
cp ~/savegame ~/savegame.reset
cat ~/savegame.txt | \
  sed -n '/CompletionState:/{N;s/\n//;s/\r//;p}' | \
  tr -s ' ' | grep "CompletionState:" | \
  sed 's/^.*Data Offset: \([0-9]*\)[^0-9].*$/\1/' | \
  while read offset
  do
    dd if=/dev/zero of=~/savegame.reset bs=1 count=1 conv=notrunc seek=$offset 2>/dev/null >/dev/null
    echo "$offset"
  done | pv -l -s 2900 > /dev/null
```

### Simplifying Rival Farming

Using the below, it should make it easier to get rivals to spawn. The process is:

1. Run all of the below to set any existing rivals to level 4, reset the cooldown on every region, and max out the region score.
2. Load the game, and then go to each region and get one rival to spawn in each region
3. Repeat.

#### Region Score Maxing

The max region score is 9611, or `0x8B250000` in little-endian uint32, and is captured as `InsurrectionScore` in the save file. We can max every region's score, assuming that the savegame has been processed (using `time python3 process_adf.py ~/savegame > ~/savegame.txt`), with:

```bash
cat ~/savegame.txt | grep -A1 "^ *InsurrectionScore:" | grep "^  *[0-9-]" | tr -d ' ' | tr '()' ':' | cut -d ':' -f4 | while read offset; do echo -en "\x8B\x25\x00\x00" | dd conv=notrunc bs=1 count=4 of=~/savegame seek=$offset; done
```

#### Rival Level Maxing

The max rival level is 4, and this is calculated at run-time based on the rival experience (which maxes at 2304, or `0x00090000`). This is captured in the (case-sensitive!) field of `XP` (note that player experience is captured in `Xp`) in the save file. Similar to above, we can max every rival with:

```bash
cat ~/savegame.txt | grep -A1 "^ *XP:" | grep "^  *[0-9-]" | tr -d ' ' | tr '()' ':' | cut -d ':' -f4 | while read offset; do echo -en "\x00\x09\x00\x00" | dd conv=notrunc bs=1 count=4 of=~/savegame seek=$offset; done
```

#### Region Rival Timer Reset

You can spawn at most one rival per region per in-game hour, so when farming this can be force a fair bit of grind. The timer for this is captured in the `Cooldown` float32 property in the save file, which we can set to 0.

```bash
cat ~/savegame.txt | grep -A1 "^ *Cooldown:" | grep "^  *[0-9-]" | tr -d ' ' | tr '()' ':' | cut -d ':' -f4 | while read offset; do echo -en "\x00\x00\x00\x00" | dd conv=notrunc bs=1 count=4 of=~/savegame seek=$offset; done
```
