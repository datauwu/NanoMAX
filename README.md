# NanoMAX

A 7-byte universal transition loop for 16-bit x86.

```asm
loop:
    movsw           ; A5
    add  si, [si]   ; 03 34
    xchg si, di     ; 87 F7
    jmp  short loop ; EB F9
```

Its machine code is:

```text
A5 03 34 87 F7 EB F9
```

NanoMAX reduces [MiniMAX][minimax]'s 8-byte loop by replacing:

```asm
lodsw
add si, ax
```

with:

```asm
add si, [si]
```

The standard MiniMAX 16-bit representation stores words as little-endian doubled two's-complement values. NanoMAX adds 2 to each stored word to account for the removed `lodsw`.

## Formal proof

This proof compares NanoMAX with the standard [MiniMAX][minimax] transition

The universality premise is the linked [MiniMAX Turing completeness proof][minimax-proof]

The simulation relation is defined at loop entry

### Machine invariant

Let MiniMAX memory be

```math
M
```

Let NanoMAX memory be

```math
N
```

Let the common source and destination offsets be

```math
SI=s
```

```math
DI=d
```

Let their common parity be

```math
s\equiv d\equiv b\pmod2
```

Assume

```math
DS=ES
```

```math
DF=0
```

For every allocated word address

```math
a\equiv b\pmod2
```

require

```math
N[a]\equiv M[a]+2\pmod{2^{16}}
```

All register arithmetic is modulo

```math
2^{16}
```

Every stored MiniMAX word is even

Therefore every stored NanoMAX word is also even

The two machines use the same allocated word addresses

### Copy step

Both machines first execute `MOVSW`

Let their resulting memories be

```math
M_1
```

and

```math
N_1
```

The MiniMAX copy writes

```math
M[s]
```

to

```math
d
```

The NanoMAX copy writes

```math
N[s]
```

to

```math
d
```

The source relation gives

```math
N[s]\equiv M[s]+2\pmod{2^{16}}
```

Therefore the encoding relation is preserved at the destination

All other words remain unchanged

Hence

```math
N_1[a]\equiv M_1[a]+2\pmod{2^{16}}
```

for every allocated word address

This includes

```math
s=d
```

It also includes the case where the destination word is read by the next arithmetic step

Because all word starts have the same parity distinct word starts differ by a multiple of two bytes

Therefore distinct words cannot partially overlap

After `MOVSW` both machines have

```math
SI=s+2
```

```math
DI=d+2
```

### Arithmetic step

Let the post-copy MiniMAX word at the next source address be

```math
w=M_1[s+2]
```

The encoding relation gives

```math
N_1[s+2]\equiv w+2\pmod{2^{16}}
```

MiniMAX executes `LODSW`

Therefore

```math
AX=w
```

and

```math
SI=s+4
```

MiniMAX then executes `ADD`

Its resulting source offset is

```math
r_M\equiv s+4+w\pmod{2^{16}}
```

NanoMAX instead executes `ADD` directly after `MOVSW`

Its resulting source offset is

```math
r_N\equiv s+2+N_1[s+2]\pmod{2^{16}}
```

Substitution gives

```math
r_N\equiv s+2+w+2\pmod{2^{16}}
```

Therefore

```math
r_N\equiv s+4+w\pmod{2^{16}}
```

Hence

```math
r_N=r_M
```

as 16-bit offsets

Because

```math
w\equiv0\pmod2
```

the common result preserves the address parity

```math
r_M\equiv r_N\equiv b\pmod2
```

### Exchange step

Both machines execute `XCHG`

After the exchange both machines have

```math
SI=d+2
```

```math
DI=r_M=r_N
```

The memory encoding relation is unchanged

The common address parity is preserved

Therefore the simulation relation holds again at loop entry

### Unobserved state

MiniMAX leaves

```math
AX=w
```

NanoMAX does not need to preserve `AX`

NanoMAX never reads `AX`

MiniMAX overwrites `AX` with `LODSW` before reading it on every iteration

Therefore `AX` is outside the simulation relation

The 7-byte loop uses `JMP`

Therefore arithmetic flags are outside the simulation relation for the universal loop

### DOS branch

The DOS variants use `JNZ`

Both `ADD` instructions produce the same 16-bit result

Therefore

```math
ZF_M=ZF_N
```

`XCHG` does not modify the flags

Therefore both DOS variants make the same branch decision

Other arithmetic flags do not need to match because neither loop reads them

### Induction

Assume the simulation relation holds at one loop entry

The copy step preserves the memory encoding

The arithmetic step produces the same control offset

The exchange step restores the loop entry register relation

Therefore the simulation relation holds at the next loop entry

By induction the relation holds for every defined iteration

Thus NanoMAX simulates MiniMAX step for step on memory `SI` and `DI`

### Unbounded model

Let a logical MiniMAX word be

```math
x
```

Define the NanoMAX encoding by

```math
E(x)=x+1
```

Its inverse is

```math
E^{-1}(y)=y-1
```

Therefore

```math
E
```

is a computable bijection

MiniMAX reaches the next logical control position by

```math
p+2+x
```

NanoMAX reaches the next logical control position by

```math
p+1+E(x)
```

Substitution gives

```math
p+1+E(x)=p+2+x
```

The copy operation preserves the same encoding

When MiniMAX extends memory NanoMAX extends the corresponding memory location and stores the encoded word

Therefore unlimited bounds extension preserves the simulation relation

The linked MiniMAX proof establishes Turing completeness by compilation from extended Smallfuck with unlimited bounds extension

Every MiniMAX computation therefore has a corresponding NanoMAX computation

Hence NanoMAX is Turing complete under the same abstract unlimited bounds model

The finite 16-bit implementation is bounded and is not itself an unbounded machine

## The 13-byte DOS interpreter code

The repository includes `NanoMAX.COM` containing the 13 executable interpreter bytes.

```asm
org 0x100

    mov  si, 0x010E
    mov  di, si

loop:
    movsw
    add  si, [si]
    xchg si, di
    jnz  loop
    ret
```

Its machine code is:

```text
BE 0E 01 89 F7 A5 03 34 87 F7 75 F9 C3
```

Its code layout is:

```text
0100  BE 0E 01    mov si,010Eh
0103  89 F7       mov di,si
0105  A5          movsw
0106  03 34       add si,[si]
0108  87 F7       xchg si,di
010A  75 F9       jnz 0105h
010C  C3          ret
```

The program begins at offset `0x010E`.

`NanoMAX.COM` itself is exactly 13 bytes.

A one-byte alignment gap is inserted after the interpreter code before the program data.

```sh
cp NanoMAX.COM program.com
printf '\x00' >> program.com
cat program.bin >> program.com
```

The resulting layout is:

```text
0100-010C  13-byte interpreter code
010D       alignment byte
010E       appended NanoMAX program
```

The interpreter assumes `DS = ES` and `DF = 0`.

The `JNZ` exit convention is the same one used by MiniMAX. A computed next location of `DS:0000` terminates the loop and reaches `RET`.

`NanoMAX.COM` has SHA-256:

```text
a66c5d166f3e05d415e944c3efadb53bea48cbe301b87e96a1cc5f352c6f332d
```

## License

CC0 1.0 Universal.

[minimax]: https://esolangs.org/wiki/MiniMAX
[minimax-proof]: https://esolangs.org/wiki/MiniMAX_Turing-completeness_proof
