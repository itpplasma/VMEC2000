# `LWOUTTXT` W7-X validation

The validation input is
`gvec-stability/benchmarks/gliss/w7x_nuehrenberg_1996/input/input.low_i_high_mirror`
with the single line `LWOUTTXT = T` added to `INDATA`. The resulting input has
SHA-256
`7f14a029fe9a9e1cb056ad660159a377ddcd106ef9b23a271c94a1bbf383b7bd`.

Starting from VMEC2000 revision
`728af8bd6c796b36a0aa85fe298e507791e57c6e`, the patched executable was built
and run with:

```sh
fo build --flag '-O3'
rm -f /mnt/storage/w7x-vmec-text/wout_low_i_high_mirror.txt
fo exec --no-build --cwd /mnt/storage/w7x-vmec-text xvmec \
    input.low_i_high_mirror
```

No OpenMP thread-count environment variable was set. VMEC terminated normally
after 32.14 seconds and wrote both output formats:

```text
a4d2edd2fd86fd23e78299254fd760dddc784efa18b5b7670127eab7b31b51f4  wout_low_i_high_mirror.txt
44fa5c68528598a0e6898356159a98fe9b9d76607967bfe972240560bfc48698  wout_low_i_high_mirror.nc
```

The text file contains 53,595 lines and begins with:

```text
VMEC VERSION = 9.0
   18.996749305329470       0.18981684306147864        0.0000000000000000                1   11.377887200000000        8.3463087999999992        1.7027601819148948
           5         145           9           7         128         238           8        1695           0           0           0
```
