---
layout: post
title: "How I cracked obscure zlib compression format of old MySQL blobs"
date: 2020-10-03 01:00:00
categories:
    - blog
tags:
    - piracy
    - puzzle
    - random
---

Looking around the web for old [pre databases\[0\]][0] I stumbled upon [this nice collection of preDBs dumps\[1\]][1]. [The one\[2\]][2] that caught my interest was shared by the user called Islander to celebrate him `quitting this world and starting a real life`. Shared in October 2014, it was built across years 2005-2011, supposedly being the biggest database of nfos, sfvs and other metadata at 2011.

7.7GB RAR archive includes 452MB SQL dump of pre entries and over 5.3GB SQL dump of nfo files. After importing these dumps into MySQL database server (8.0.21 hosted in Docker) I discovered that something is off with `nfo` schema. 

<!--break-->

To briefly give an idea on how the table looks like:

```
> SELECT COLUMN_NAME, ORDINAL_POSITION, COLUMNT_TYPE FROM information_schema.columns where TABLE_NAME='nfo'

<
COLUMN_NAME	ORDINAL_POSITION    COLUMN_TYPE
id              1	            int unsigned
rel_name        2                   varchar
rel_nfo         3                   longblog
rel_filename    4                   varchar	
```

Cool. Let's snatch one of those blobs:
```
> SELECT CONVERT(rel_nfo using ascii) from nfo where id=85443

< "$x????v??????M???b?J????&n?M?R????a$?bM?*E??...
```

Huh? Turns out blobs are compressed. Since it's MySQL, my first guess is of course **zlib**.

Trying to decompress the file using Python `zlib` module:
```
>>> import zlib
>>> with open('./nfo-rel_nfo.bin', 'rb') as file:
...     zlib.decompress(file)
... 
Traceback (most recent call last):
  File "<stdin>", line 2, in <module>
zlib.error: Error -3 while decompressing data: incorrect header check
```

No luck. Maybe it's not zlib after all? I decide to verify that using [binwalk\[3\]][3], a tool for heuristic binary-level analysis.
```
$ binwalk nfo-rel_nfo.bin

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
4             0x4             Zlib compressed data, default compression
```

It detects zlib signature at `0x04` byte. How does it look like?

```
$ xxd nfo-rel_nfo.bin | head -1
00000000: 2224 0000 789c a599 dd76 dbb8 1180 eff5  "$..x....v......
```

Perhaps it's worth take a look at other file?

```
xxd nfo-rel_nfo-0.bin | head -1
00000000: 314e 0000 789c ed5c dd72 dbc6 92be dfaa  1N..x..\.r......
```

After confirming it with numerous other files, it seems all of them share the same `0x4` and `0x5` bytes. What does it mean in terms of zlib compression?

My first steps lead me to [RFC 1950\[4\]][4] stating, that:
- `0x0` defines CMF (Compression Method and flags)
- `0x1` defines FLG (FLaGs)
- from `0x2` to `0x5` represent DICT if flag FDICT was set on 5th bit of `0x1`
- everything beyond that point is compressed data
- files ends with Adler-32 checksum

Considering that all the files share the same 4th and 5th byte, that are supposed to be either part of compression stream or FDICT while `0x0` and `0x1` are always different, it doesn't make much of a sense.

After hours of searching, I found [this stackoverflow question\[5\]][5] asked by someone struggling with a similar problem. Upgrading zlib in some networking library broke the mutual compatibility between resources using old and new version of that library.

They noticed that old zlib library (possibly 1.1.3) returns:
```
00000000: 1f00 0000 789c 0bc9 c82c 5600 a244 8592  ....x....,V..D..
```

While 1.2.11 returns:
```
00000000: 78da 0bc9 c82c 5600 a244 8592 d4e2 1285  x....,V..D......
```

Supposing the shared part starting with `0x0bc9 c82c 5600 a244...` is compressed data stream, `0x789c` are flags for 1.1.3 zlib compression and `0x78da` are flags set by zlib 1.2.11. However, old zlib adds a little extra to the output - `0x1f000000`. What's the meaning behind that?

That person eventually found the answer on the same day. Citing:
```
Turns out the older zlib - or possibly the older networking library - had a 32-bit "uncompressed output" size prepended to its compressed output.
```

Let's checkout that theory! Coincidentally, yet luckily I'm in possession of uncompressed version of this nfo file acquired somewhere else.
```
$ ls -l 00-va_-_big_dance_party_vol_1-\(185_312\)-1994-zzzz.nfo 
-rw-r--r-- 1 max max 9250 Mar  9 01:02 '00-va_-_big_dance_party_vol_1-(185_312)-1994-zzzz.nfo'
```

9250 bytes. Going back to that strange signature, first 32 bits are: `0x22240000`, `572784640` converted to decimal. Not even close to `9250`. It took me a few moments to realize my mistake. Of course it needs to be read in little-endian order! After swapping - `0x00002422` is `9250`! Gotcha!!

Let's try removing first 32 bits of the file to fix flags offset and see what happens:
```
>>> with open('./nfo-rel_nfo.bin', 'rb') as input, open('./out.nfo', 'wb') as output:
...     x = zlib.decompress(input.read())
...     #  no more errors yay! :D
...     output.write(x)
... 
9250
```

It seems it worked. Let's check out the output:
![Screenshot showing decompressed file]({{ site.baseurl }}images/2020/zzzz.png)

Victory! Now, in order to extract all the nfos from the database, I wrote this nice little Python script:
```
from pathlib import Path
import zlib

import mysql.connector
from tqdm import tqdm


if __name__ == '__main__':

    dump_path = '/home/max/Projects/pre_archive/islander'

    cnx = mysql.connector.connect(user='root', password='root',
                                  host='localhost', database='islander')
    cursor = cnx.cursor()

    data_query = 'SELECT rel_name, rel_filename, rel_nfo FROM nfo;'

    count_query = 'SELECT COUNT(*) FROM nfo;'
    cursor.execute(count_query)
    nfo_count = cursor.fetchone()

    cursor.execute(data_query)

    with tqdm(total=nfo_count[0]) as pbar:
        for (rel_name, rel_filename, rel_nfo) in cursor:
            release_path = f'{dump_path}/{rel_name}'
            filename = f'{rel_name}.nfo'.replace('/', '')
            Path(release_path).mkdir(parents=True, exist_ok=True)

            with open(f'{release_path}/{filename}', 'wb') as nfo_file:
                try:
                    d = zlib.decompress(rel_nfo[4:])  # trim 32 header and decompress
                    nfo_file.write(d)
                    nfo_file.close()
                except zlib.error:
                    print(f'Error decompressing: {rel_name}')

            pbar.update()

    cursor.close()
    cnx.close()

```

Crucial part is the line 32, where binary data is being read by `zlib.decompress` omitting first 4 bytes. After adding some precautious steps to the script (some filenames has forbidden characters, plus ext4 doesn't handle 72,923 files in a single directory well) I was able to decompress all the blobs in this database 🎉🥰
## Links
~~~
[0]: https://en.wikipedia.org/wiki/Nuke_(warez)#Pre_database
[1]: http://rescene.wikidot.com/scene-databases
[2]: http://rescene.wikidot.com/dump-islander
[3]: https://github.com/ReFirmLabs/binwalk
[4]: https://tools.ietf.org/html/rfc1950
[5]: https://stackoverflow.com/q/59494980/6158307
~~~

[0]: https://en.wikipedia.org/wiki/Nuke_(warez)#Pre_database
[1]: http://rescene.wikidot.com/scene-databases
[2]: http://rescene.wikidot.com/dump-islander
[3]: https://github.com/ReFirmLabs/binwalk
[4]: https://tools.ietf.org/html/rfc1950
[5]: https://stackoverflow.com/q/59494980/6158307