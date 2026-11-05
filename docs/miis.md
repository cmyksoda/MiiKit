# Reading and writing Miis

`mii.h` and `mii.c` are MiiKit's data half.

Each call that can fail returns 0 or more, or a negative `MII_E_*` code
(`mii_strerror()` names it). The snippets below need:

```c
#include <stdio.h>
#include <string.h>
#include <gccore.h>
#include <ogc/lwp_watchdog.h>   // gettime()
#include <wiiuse/wpad.h>
#include "mii.h"
```

## Read every Mii

```c
mii_db db = { 0 };
mii m;
char name[MII_NAME_UTF8_MAX];

int r = mii_db_load(&db);      // MII_E_NO_FILE: the Mii Channel has never been opened
if (r < 0) {
    printf("%s\n", mii_strerror(r));
    return;
}
for (int i = 0; i < MII_DB_SLOTS; ++i)
    if (mii_db_get(&db, i, &m) == MII_OK) {
        mii_get_name(m.name, name, sizeof(name));
        printf("%d: %s, likes %s\n", i, name, mii_color_name(m.fav_color));
    }
mii_db_free(&db);
```

## Make a Mii and add it to the plaza

```c
uint32_t seed = (uint32_t) gettime();
mii m;
const char *why;

mii_random(&m, MII_ANY, MII_ANY, &seed);   // or mii_default(&m, MII_FEMALE)
mii_set_name(m.name, "Homebrew");
mii_set_name(m.creator, "My App");
m.height = 90;               // 0 shortest .. 128 tallest (64 the middle)
m.build = 40;                // 0 thinnest .. 128 heaviest
m.birth_month = 10;          // 0/0 for no birthday
m.birth_day = 24;
m.fav_color = MII_GREEN;
m.favorite = 1;              // starred in the Mii Channel
m.local_only = 0;            // 0: Mingle on
if (mii_make_id(&m) < 0 || mii_check(&m, &why) != MII_OK)
    return;                  // why says what the Wii would refuse

mii_db db = { 0 };
if (mii_db_load(&db) == MII_OK) {
    int slot = mii_db_add(&db, &m);         // the first free slot, or MII_E_FULL
    int r = slot < 0 ? slot : mii_db_save(&db);
    if (r == MII_E_IO || r == MII_E_VERIFY)
        r = mii_db_save(&db);               // a save that did not finish may be retried
    // MII_E_CHANGED: someone else wrote the file; load it again and add again
    mii_db_free(&db);
}
```

## Change a Mii

```c
mii_db db = { 0 };
mii m;

if (mii_db_load(&db) == MII_OK && mii_db_get(&db, 3, &m) == MII_OK) {
    m.height = 128;
    m.local_only = 1;        // Mingle off
    if (mii_db_put(&db, 3, &m) == MII_OK)
        mii_db_save(&db);
}
mii_db_free(&db);
```

`mii_db_put` stores a copy. It makes a Mii made on another Wii local only,
as RFL does, so read the slot back to see what was stored.

## A .mii file

```c
uint8_t buf[MII_STORE_SIZE + 1];
FILE *f = fopen("sd:/mii/friend.mii", "rb");   // after fatInitDefault()
size_t n = f ? fread(buf, 1, sizeof(buf), f) : 0;
mii m;

if (f)
    fclose(f);
if (mii_import(buf, n, &m) == MII_OK && mii_check(&m, NULL) == MII_OK)
    ;                        // add it as above (keep its creation ID)

uint8_t raw[MII_SIZE], with_crc[MII_STORE_SIZE];
mii_pack(&m, raw);           // the 74-byte .mii
mii_store(raw, with_crc);    // the 76-byte form
```

## A Mii on Wii Remote 1

```c
WPAD_Init();                 // once, early: a Remote connects a few seconds later
u64 end = gettime() + secs_to_ticks(10);
while (WPAD_Probe(0, NULL) != WPAD_ERR_NONE && gettime() < end)
    usleep(100 * 1000);

mii_remote rm;
if (mii_remote_read(0, &rm) == MII_OK) {   // about 1 s; MII_E_FORMAT: leave it alone
    int slot = mii_remote_free_slot(&rm);  // never write over the user's Miis
    if (slot >= 0 && mii_remote_put(&rm, slot, &m) == MII_OK)
        mii_remote_write(0, &rm);          // about 8 s: both copies, each read back
}
```

If `mii_db_isolated()` says the Wii has mingling off, set `m.local_only = 1`
before putting it on a Remote, as RFL does.

## A Mii on a Message Board letter

This uses hbc-reborn's [sdk/mboard](https://github.com/Monsterray/hbc-reborn/tree/master/sdk/mboard), which isn't part of MiiKit.

```c
#include "mboard.h"                        // sdk/mboard

uint8_t face[MII_SIZE];
mii_pack(&m, face);                        // m passed mii_check, or the icon is dropped
mboard_msg msg = { 0 };
msg.title = "My App";
msg.body = "Hello from my Mii.";
msg.face = face;
mboard_post_msg(&msg);
```

## Errors

| Code | What to do |
| --- | --- |
| `MII_E_NO_FILE` | No database: the Mii Channel has never been opened. The library does not create one. |
| `MII_E_FORMAT` | The database or a Remote's set is not as the Wii makes it. Leave it alone: nothing was written. |
| `MII_E_CHANGED` | Someone else saved the database since you loaded it. Load it again and make the change again. |
| `MII_E_IO`, `MII_E_VERIFY` | A save did not finish. Call `mii_db_save` again. |
| `MII_E_INVALID` | The Mii fails the Wii's check: `mii_check(&m, &why)` says why. |
| `MII_E_EXISTS` | That creation ID is in another slot. |
| `MII_E_FULL` | No free slot. |
| `MII_E_NO_REMOTE`, `MII_E_TIMEOUT` | Wake the Remote and try again. libogc drops a Remote left idle for 5 minutes. |
| `MII_E_UNSUPPORTED` | The Remote calls need devkitPro's libogc 3.x. |
