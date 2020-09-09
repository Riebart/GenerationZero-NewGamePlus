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

## Building on Ubuntu 20

### Setting up `deca`

```bash
apt update
apt install build-essential git python3-pip python3-numpy python3-matplotlib
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
