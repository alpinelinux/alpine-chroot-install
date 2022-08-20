#!/bin/sh
# vim: set ts=4:
#---help---
# Usage: alpine-chroot-install [options]
#
# This script installs Alpine Linux into a chroot and optionally sets up
# qemu-user and binfmt to emulate different architecture (e.g. armhf).
#
# If qemu-user and binfmt is needed, the script checks if both are available.
# If not, it tries to install them using apt-get. Beside this the script should
# work on any Linux system.
#
# It also creates script "enter-chroot" inside the chroot directory, that may
# be used to enter the chroot environment. That script do the following:
#
#   1. saves environment variables specified by $CHROOT_KEEP_VARS and PWD,
#   2. chroots into $CHROOT_DIR,
#   3. starts clean environment using "env -i",
#   4. switches user and simulates full login using "su -l",
#   5. loads saved environment variables and changes directory to saved PWD,
#   6. executes specified command or "sh" if not provided.
#
# Example:
#   sudo alpine-chroot-install -d /alpine -p build-base -p cmake
#	/alpine/enter-chroot -u $USER ./build
#
# Options and environment variables:
#   -a ARCH                CPU architecture for the chroot. If not set, then it's
#                          the same as the host's architecture. If it's different
#                          from the host's architecture, then it will be emulated
#                          using qemu-user. Options: x86_64, x86, aarch64, armhf,
#                          armv7, ppc64le, riscv64, s390x.
#
#   -b ALPINE_BRANCH       Alpine branch to install (default is latest-stable).
#
#   -d CHROOT_DIR          Absolute path to the directory where Alpine chroot
#                          should be installed (default is /alpine).
#
#   -i BIND_DIR            Absolute path to the directory on the host system that
#                          should be mounted on the same path inside the chroot
#                          (default is PWD, if it's under /home, or none).
#
#   -k CHROOT_KEEP_VARS... Names of the environment variables to pass from the
#                          host environment into chroot by the enter-chroot
#                          script. Name may be an extended regular expression.
#                          Default: ARCH CI QEMU_EMULATOR TRAVIS_.*.
#
#   -m ALPINE_MIRROR...    URI of the Aports mirror to fetch packages from
#                          (default is http://dl-cdn.alpinelinux.org/alpine).
#
#   -p ALPINE_PACKAGES...  Alpine packages to install into the chroot (default is
#                          build-base ca-certificates ssl_client).
#
#   -r EXTRA_REPOS...      Alpine repositories to be added to
#                          /etc/apk/repositories (main and community from
#                          $ALPINE_MIRROR are always added).
#
#   -t TEMP_DIR            Absolute path to the directory where to store temporary
#                          files (defaults to `mktemp -d`).
#
#   -h                     Show this help message and exit.
#
#   -v                     Print version and exit.
#
#   APK_TOOLS_URI          URL of apk.static to download. Default is x86_64
#                          binary from
#                          https://gitlab.alpinelinux.org/alpine/apk-tools/-/packages.
#
#   APK_TOOLS_SHA256       SHA-256 checksum of $APK_TOOLS_URI.
#
# Each option can be also provided by environment variable. If both option and
# variable is specified and the option accepts only one argument, then the
# option takes precedence.
#
# https://github.com/alpinelinux/alpine-chroot-install
#---help---
set -eu

#=======================  C o n s t a n t s  =======================#

: ${APK_TOOLS_URI:="https://gitlab.alpinelinux.org/api/v4/projects/5/packages/generic/v2.12.10/x86_64/apk.static"}
: ${APK_TOOLS_SHA256:="d7506bb11327b337960910daffed75aa289d8bb350feab624c52965be82ceae8"}

# Alpine APK keys for packages verification.
ALPINE_KEYS='
4a6a0840:MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA1yHJxQgsHQREclQu4Ohe\nqxTxd1tHcNnvnQTu/UrTky8wWvgXT+jpveroeWWnzmsYlDI93eLI2ORakxb3gA2O\nQ0Ry4ws8vhaxLQGC74uQR5+/yYrLuTKydFzuPaS1dK19qJPXB8GMdmFOijnXX4SA\njixuHLe1WW7kZVtjL7nufvpXkWBGjsfrvskdNA/5MfxAeBbqPgaq0QMEfxMAn6/R\nL5kNepi/Vr4S39Xvf2DzWkTLEK8pcnjNkt9/aafhWqFVW7m3HCAII6h/qlQNQKSo\nGuH34Q8GsFG30izUENV9avY7hSLq7nggsvknlNBZtFUcmGoQrtx3FmyYsIC8/R+B\nywIDAQAB
5243ef4b:MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAvNijDxJ8kloskKQpJdx+\nmTMVFFUGDoDCbulnhZMJoKNkSuZOzBoFC94omYPtxnIcBdWBGnrm6ncbKRlR+6oy\nDO0W7c44uHKCFGFqBhDasdI4RCYP+fcIX/lyMh6MLbOxqS22TwSLhCVjTyJeeH7K\naA7vqk+QSsF4TGbYzQDDpg7+6aAcNzg6InNePaywA6hbT0JXbxnDWsB+2/LLSF2G\nmnhJlJrWB1WGjkz23ONIWk85W4S0XB/ewDefd4Ly/zyIciastA7Zqnh7p3Ody6Q0\nsS2MJzo7p3os1smGjUF158s6m/JbVh4DN6YIsxwl2OjDOz9R0OycfJSDaBVIGZzg\ncQIDAQAB
524d27bb:MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAr8s1q88XpuJWLCZALdKj\nlN8wg2ePB2T9aIcaxryYE/Jkmtu+ZQ5zKq6BT3y/udt5jAsMrhHTwroOjIsF9DeG\ne8Y3vjz+Hh4L8a7hZDaw8jy3CPag47L7nsZFwQOIo2Cl1SnzUc6/owoyjRU7ab0p\niWG5HK8IfiybRbZxnEbNAfT4R53hyI6z5FhyXGS2Ld8zCoU/R4E1P0CUuXKEN4p0\n64dyeUoOLXEWHjgKiU1mElIQj3k/IF02W89gDj285YgwqA49deLUM7QOd53QLnx+\nxrIrPv3A+eyXMFgexNwCKQU9ZdmWa00MjjHlegSGK8Y2NPnRoXhzqSP9T9i2HiXL\nVQIDAQAB
5261cecb:MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAwlzMkl7b5PBdfMzGdCT0\ncGloRr5xGgVmsdq5EtJvFkFAiN8Ac9MCFy/vAFmS8/7ZaGOXoCDWbYVLTLOO2qtX\nyHRl+7fJVh2N6qrDDFPmdgCi8NaE+3rITWXGrrQ1spJ0B6HIzTDNEjRKnD4xyg4j\ng01FMcJTU6E+V2JBY45CKN9dWr1JDM/nei/Pf0byBJlMp/mSSfjodykmz4Oe13xB\nCa1WTwgFykKYthoLGYrmo+LKIGpMoeEbY1kuUe04UiDe47l6Oggwnl+8XD1MeRWY\nsWgj8sF4dTcSfCMavK4zHRFFQbGp/YFJ/Ww6U9lA3Vq0wyEI6MCMQnoSMFwrbgZw\nwwIDAQAB
58199dcc:MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA3v8/ye/V/t5xf4JiXLXa\nhWFRozsnmn3hobON20GdmkrzKzO/eUqPOKTpg2GtvBhK30fu5oY5uN2ORiv2Y2ht\neLiZ9HVz3XP8Fm9frha60B7KNu66FO5P2o3i+E+DWTPqqPcCG6t4Znk2BypILcit\nwiPKTsgbBQR2qo/cO01eLLdt6oOzAaF94NH0656kvRewdo6HG4urbO46tCAizvCR\nCA7KGFMyad8WdKkTjxh8YLDLoOCtoZmXmQAiwfRe9pKXRH/XXGop8SYptLqyVVQ+\ntegOD9wRs2tOlgcLx4F/uMzHN7uoho6okBPiifRX+Pf38Vx+ozXh056tjmdZkCaV\naQIDAQAB
58cbb476:MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAoSPnuAGKtRIS5fEgYPXD\n8pSGvKAmIv3A08LBViDUe+YwhilSHbYXUEAcSH1KZvOo1WT1x2FNEPBEFEFU1Eyc\n+qGzbA03UFgBNvArurHQ5Z/GngGqE7IarSQFSoqewYRtFSfp+TL9CUNBvM0rT7vz\n2eMu3/wWG+CBmb92lkmyWwC1WSWFKO3x8w+Br2IFWvAZqHRt8oiG5QtYvcZL6jym\nY8T6sgdDlj+Y+wWaLHs9Fc+7vBuyK9C4O1ORdMPW15qVSl4Lc2Wu1QVwRiKnmA+c\nDsH/m7kDNRHM7TjWnuj+nrBOKAHzYquiu5iB3Qmx+0gwnrSVf27Arc3ozUmmJbLj\nzQIDAQAB
58e4f17d:MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAvBxJN9ErBgdRcPr5g4hV\nqyUSGZEKuvQliq2Z9SRHLh2J43+EdB6A+yzVvLnzcHVpBJ+BZ9RV30EM9guck9sh\nr+bryZcRHyjG2wiIEoduxF2a8KeWeQH7QlpwGhuobo1+gA8L0AGImiA6UP3LOirl\nI0G2+iaKZowME8/tydww4jx5vG132JCOScMjTalRsYZYJcjFbebQQolpqRaGB4iG\nWqhytWQGWuKiB1A22wjmIYf3t96l1Mp+FmM2URPxD1gk/BIBnX7ew+2gWppXOK9j\n1BJpo0/HaX5XoZ/uMqISAAtgHZAqq+g3IUPouxTphgYQRTRYpz2COw3NF43VYQrR\nbQIDAQAB
60ac2099:MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAwR4uJVtJOnOFGchnMW5Y\nj5/waBdG1u5BTMlH+iQMcV5+VgWhmpZHJCBz3ocD+0IGk2I68S5TDOHec/GSC0lv\n6R9o6F7h429GmgPgVKQsc8mPTPtbjJMuLLs4xKc+viCplXc0Nc0ZoHmCH4da6fCV\ntdpHQjVe6F9zjdquZ4RjV6R6JTiN9v924dGMAkbW/xXmamtz51FzondKC52Gh8Mo\n/oA0/T0KsCMCi7tb4QNQUYrf+Xcha9uus4ww1kWNZyfXJB87a2kORLiWMfs2IBBJ\nTmZ2Fnk0JnHDb8Oknxd9PvJPT0mvyT8DA+KIAPqNvOjUXP4bnjEHJcoCP9S5HkGC\nIQIDAQAB
6165ee59:MIICIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEAutQkua2CAig4VFSJ7v54\nALyu/J1WB3oni7qwCZD3veURw7HxpNAj9hR+S5N/pNeZgubQvJWyaPuQDm7PTs1+\ntFGiYNfAsiibX6Rv0wci3M+z2XEVAeR9Vzg6v4qoofDyoTbovn2LztaNEjTkB+oK\ntlvpNhg1zhou0jDVYFniEXvzjckxswHVb8cT0OMTKHALyLPrPOJzVtM9C1ew2Nnc\n3848xLiApMu3NBk0JqfcS3Bo5Y2b1FRVBvdt+2gFoKZix1MnZdAEZ8xQzL/a0YS5\nHd0wj5+EEKHfOd3A75uPa/WQmA+o0cBFfrzm69QDcSJSwGpzWrD1ScH3AK8nWvoj\nv7e9gukK/9yl1b4fQQ00vttwJPSgm9EnfPHLAtgXkRloI27H6/PuLoNvSAMQwuCD\nhQRlyGLPBETKkHeodfLoULjhDi1K2gKJTMhtbnUcAA7nEphkMhPWkBpgFdrH+5z4\nLxy+3ek0cqcI7K68EtrffU8jtUj9LFTUC8dERaIBs7NgQ/LfDbDfGh9g6qVj1hZl\nk9aaIPTm/xsi8v3u+0qaq7KzIBc9s59JOoA8TlpOaYdVgSQhHHLBaahOuAigH+VI\nisbC9vmqsThF2QdDtQt37keuqoda2E6sL7PUvIyVXDRfwX7uMDjlzTxHTymvq2Ck\nhtBqojBnThmjJQFgZXocHG8CAwEAAQ==
61666e3f:MIICIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEAlEyxkHggKCXC2Wf5Mzx4\nnZLFZvU2bgcA3exfNPO/g1YunKfQY+Jg4fr6tJUUTZ3XZUrhmLNWvpvSwDS19ZmC\nIXOu0+V94aNgnhMsk9rr59I8qcbsQGIBoHzuAl8NzZCgdbEXkiY90w1skUw8J57z\nqCsMBydAueMXuWqF5nGtYbi5vHwK42PffpiZ7G5Kjwn8nYMW5IZdL6ZnMEVJUWC9\nI4waeKg0yskczYDmZUEAtrn3laX9677ToCpiKrvmZYjlGl0BaGp3cxggP2xaDbUq\nqfFxWNgvUAb3pXD09JM6Mt6HSIJaFc9vQbrKB9KT515y763j5CC2KUsilszKi3mB\nHYe5PoebdjS7D1Oh+tRqfegU2IImzSwW3iwA7PJvefFuc/kNIijfS/gH/cAqAK6z\nbhdOtE/zc7TtqW2Wn5Y03jIZdtm12CxSxwgtCF1NPyEWyIxAQUX9ACb3M0FAZ61n\nfpPrvwTaIIxxZ01L3IzPLpbc44x/DhJIEU+iDt6IMTrHOphD9MCG4631eIdB0H1b\n6zbNX1CXTsafqHRFV9XmYYIeOMggmd90s3xIbEujA6HKNP/gwzO6CDJ+nHFDEqoF\nSkxRdTkEqjTjVKieURW7Swv7zpfu5PrsrrkyGnsRrBJJzXlm2FOOxnbI2iSL1B5F\nrO5kbUxFeZUIDq+7Yv4kLWcCAwEAAQ==
616a9724:MIICIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEAnC+bR4bHf/L6QdU4puhQ\ngl1MHePszRC38bzvVFDUJsmCaMCL2suCs2A2yxAgGb9pu9AJYLAmxQC4mM3jNqhg\n/E7yuaBbek3O02zN/ctvflJ250wZCy+z0ZGIp1ak6pu1j14IwHokl9j36zNfGtfv\nADVOcdpWITFFlPqwq1qt/H3UsKVmtiF3BNWWTeUEQwKvlU8ymxgS99yn0+4OPyNT\nL3EUeS+NQJtDS01unau0t7LnjUXn+XIneWny8bIYOQCuVR6s/gpIGuhBaUqwaJOw\n7jkJZYF2Ij7uPb4b5/R3vX2FfxxqEHqssFSg8FFUNTZz3qNZs0CRVyfA972g9WkJ\nhPfn31pQYil4QGRibCMIeU27YAEjXoqfJKEPh4UWMQsQLrEfdGfb8VgwrPbniGfU\nL3jKJR3VAafL9330iawzVQDlIlwGl6u77gEXMl9K0pfazunYhAp+BMP+9ot5ckK+\nosmrqj11qMESsAj083GeFdfV3pXEIwUytaB0AKEht9DbqUfiE/oeZ/LAXgySMtVC\nsbC4ESmgVeY2xSBIJdDyUap7FR49GGrw0W49NUv9gRgQtGGaNVQQO9oGL2PBC41P\niWF9GLoX30HIz1P8PF/cZvicSSPkQf2Z6TV+t0ebdGNS5DjapdnCrq8m9Z0pyKsQ\nuxAL2a7zX8l5i1CZh1ycUGsCAwEAAQ==
616abc23:MIICIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEA0MfCDrhODRCIxR9Dep1s\neXafh5CE5BrF4WbCgCsevyPIdvTeyIaW4vmO3bbG4VzhogDZju+R3IQYFuhoXP5v\nY+zYJGnwrgz3r5wYAvPnLEs1+dtDKYOgJXQj+wLJBW1mzRDL8FoRXOe5iRmn1EFS\nwZ1DoUvyu7/J5r0itKicZp3QKED6YoilXed+1vnS4Sk0mzN4smuMR9eO1mMCqNp9\n9KTfRDHTbakIHwasECCXCp50uXdoW6ig/xUAFanpm9LtK6jctNDbXDhQmgvAaLXZ\nLvFqoaYJ/CvWkyYCgL6qxvMvVmPoRv7OPcyni4xR/WgWa0MSaEWjgPx3+yj9fiMA\n1S02pFWFDOr5OUF/O4YhFJvUCOtVsUPPfA/Lj6faL0h5QI9mQhy5Zb9TTaS9jB6p\nLw7u0dJlrjFedk8KTJdFCcaGYHP6kNPnOxMylcB/5WcztXZVQD5WpCicGNBxCGMm\nW64SgrV7M07gQfL/32QLsdqPUf0i8hoVD8wfQ3EpbQzv6Fk1Cn90bZqZafg8XWGY\nwddhkXk7egrr23Djv37V2okjzdqoyLBYBxMz63qQzFoAVv5VoY2NDTbXYUYytOvG\nGJ1afYDRVWrExCech1mX5ZVUB1br6WM+psFLJFoBFl6mDmiYt0vMYBddKISsvwLl\nIJQkzDwtXzT2cSjoj3T5QekCAwEAAQ==
616ac3bc:MIICIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEAvaaoSLab+IluixwKV5Od\n0gib2YurjPatGIbn5Ov2DLUFYiebj2oJINXJSwUOO+4WcuHFEqiL/1rya+k5hLZt\nhnPL1tn6QD4rESznvGSasRCQNT2vS/oyZbTYJRyAtFkEYLlq0t3S3xBxxHWuvIf0\nqVxVNYpQWyM3N9RIeYBR/euXKJXileSHk/uq1I5wTC0XBIHWcthczGN0m9wBEiWS\n0m3cnPk4q0Ea8mUJ91Rqob19qETz6VbSPYYpZk3qOycjKosuwcuzoMpwU8KRiMFd\n5LHtX0Hx85ghGsWDVtS0c0+aJa4lOMGvJCAOvDfqvODv7gKlCXUpgumGpLdTmaZ8\n1RwqspAe3IqBcdKTqRD4m2mSg23nVx2FAY3cjFvZQtfooT7q1ItRV5RgH6FhQSl7\n+6YIMJ1Bf8AAlLdRLpg+doOUGcEn+pkDiHFgI8ylH1LKyFKw+eXaAml/7DaWZk1d\ndqggwhXOhc/UUZFQuQQ8A8zpA13PcbC05XxN2hyP93tCEtyynMLVPtrRwDnHxFKa\nqKzs3rMDXPSXRn3ZZTdKH3069ApkEjQdpcwUh+EmJ1Ve/5cdtzT6kKWCjKBFZP/s\n91MlRrX2BTRdHaU5QJkUheUtakwxuHrdah2F94lRmsnQlpPr2YseJu6sIE+Dnx4M\nCfhdVbQL2w54R645nlnohu8CAwEAAQ==
616adfeb:MIICIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEAq0BFD1D4lIxQcsqEpQzU\npNCYM3aP1V/fxxVdT4DWvSI53JHTwHQamKdMWtEXetWVbP5zSROniYKFXd/xrD9X\n0jiGHey3lEtylXRIPxe5s+wXoCmNLcJVnvTcDtwx/ne2NLHxp76lyc25At+6RgE6\nADjLVuoD7M4IFDkAsd8UQ8zM0Dww9SylIk/wgV3ZkifecvgUQRagrNUdUjR56EBZ\nraQrev4hhzOgwelT0kXCu3snbUuNY/lU53CoTzfBJ5UfEJ5pMw1ij6X0r5S9IVsy\nKLWH1hiO0NzU2c8ViUYCly4Fe9xMTFc6u2dy/dxf6FwERfGzETQxqZvSfrRX+GLj\n/QZAXiPg5178hT/m0Y3z5IGenIC/80Z9NCi+byF1WuJlzKjDcF/TU72zk0+PNM/H\nKuppf3JT4DyjiVzNC5YoWJT2QRMS9KLP5iKCSThwVceEEg5HfhQBRT9M6KIcFLSs\nmFjx9kNEEmc1E8hl5IR3+3Ry8G5/bTIIruz14jgeY9u5jhL8Vyyvo41jgt9sLHR1\n/J1TxKfkgksYev7PoX6/ZzJ1ksWKZY5NFoDXTNYUgzFUTOoEaOg3BAQKadb3Qbbq\nXIrxmPBdgrn9QI7NCgfnAY3Tb4EEjs3ON/BNyEhUENcXOH6I1NbcuBQ7g9P73kE4\nVORdoc8MdJ5eoKBpO8Ww8HECAwEAAQ==
616ae350:MIICIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEAyduVzi1mWm+lYo2Tqt/0\nXkCIWrDNP1QBMVPrE0/ZlU2bCGSoo2Z9FHQKz/mTyMRlhNqTfhJ5qU3U9XlyGOPJ\npiM+b91g26pnpXJ2Q2kOypSgOMOPA4cQ42PkHBEqhuzssfj9t7x47ppS94bboh46\nxLSDRff/NAbtwTpvhStV3URYkxFG++cKGGa5MPXBrxIp+iZf9GnuxVdST5PGiVGP\nODL/b69sPJQNbJHVquqUTOh5Ry8uuD2WZuXfKf7/C0jC/ie9m2+0CttNu9tMciGM\nEyKG1/Xhk5iIWO43m4SrrT2WkFlcZ1z2JSf9Pjm4C2+HovYpihwwdM/OdP8Xmsnr\nDzVB4YvQiW+IHBjStHVuyiZWc+JsgEPJzisNY0Wyc/kNyNtqVKpX6dRhMLanLmy+\nf53cCSI05KPQAcGj6tdL+D60uKDkt+FsDa0BTAobZ31OsFVid0vCXtsbplNhW1IF\nHwsGXBTVcfXg44RLyL8Lk/2dQxDHNHzAUslJXzPxaHBLmt++2COa2EI1iWlvtznk\nOk9WP8SOAIj+xdqoiHcC4j72BOVVgiITIJNHrbppZCq6qPR+fgXmXa+sDcGh30m6\n9Wpbr28kLMSHiENCWTdsFij+NQTd5S47H7XTROHnalYDuF1RpS+DpQidT5tUimaT\nJZDr++FjKrnnijbyNF8b98UCAwEAAQ==
616db30d:MIICIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEAnpUpyWDWjlUk3smlWeA0\nlIMW+oJ38t92CRLHH3IqRhyECBRW0d0aRGtq7TY8PmxjjvBZrxTNDpJT6KUk4LRm\na6A6IuAI7QnNK8SJqM0DLzlpygd7GJf8ZL9SoHSH+gFsYF67Cpooz/YDqWrlN7Vw\ntO00s0B+eXy+PCXYU7VSfuWFGK8TGEv6HfGMALLjhqMManyvfp8hz3ubN1rK3c8C\nUS/ilRh1qckdbtPvoDPhSbTDmfU1g/EfRSIEXBrIMLg9ka/XB9PvWRrekrppnQzP\nhP9YE3x/wbFc5QqQWiRCYyQl/rgIMOXvIxhkfe8H5n1Et4VAorkpEAXdsfN8KSVv\nLSMazVlLp9GYq5SUpqYX3KnxdWBgN7BJoZ4sltsTpHQ/34SXWfu3UmyUveWj7wp0\nx9hwsPirVI00EEea9AbP7NM2rAyu6ukcm4m6ATd2DZJIViq2es6m60AE6SMCmrQF\nwmk4H/kdQgeAELVfGOm2VyJ3z69fQuywz7xu27S6zTKi05Qlnohxol4wVb6OB7qG\nLPRtK9ObgzRo/OPumyXqlzAi/Yvyd1ZQk8labZps3e16bQp8+pVPiumWioMFJDWV\nGZjCmyMSU8V6MB6njbgLHoyg2LCukCAeSjbPGGGYhnKLm1AKSoJh3IpZuqcKCk5C\n8CM1S15HxV78s9dFntEqIokCAwEAAQ==
'

# Version of alpine-chroot-install script.
VERSION='0.14.0'


#=======================  F u n c t i o n s  =======================#

die() {
	printf '\033[1;31mERROR:\033[0m %s\n' "$@" >&2  # bold red
	exit 1
}

einfo() {
	printf '\n\033[1;36m> %s\033[0m\n' "$@" >&2  # bold cyan
}

ewarn() {
	printf '\033[1;33m> %s\033[0m\n' "$@" >&2  # bold yellow
}

# Writes Alpine APK keys embedded in this script into directory $1.
dump_alpine_keys() {
	local dest_dir="$1"
	local content id line

	mkdir -p "$dest_dir"
	for line in $ALPINE_KEYS; do
		id=${line%%:*}
		content=${line#*:}

		printf -- "-----BEGIN PUBLIC KEY-----\n$content\n-----END PUBLIC KEY-----\n" \
			> "$dest_dir/alpine-devel@lists.alpinelinux.org-$id.rsa.pub"
	done
}

normalize_arch() {
	case "$1" in
		x86 | i[3456]86) echo 'i386';;
		armhf | armv[4-9]) echo 'arm';;
		*) echo "$1";;
	esac
}

fetch_url() {
	local url="$1"

	echo "fetch $url" >&2
	if command -v curl >/dev/null; then
		curl --remote-name --connect-timeout 10 -fsSL "$url"
	elif command -v wget >/dev/null; then
		wget -T 10 --no-verbose "$url"
	else
		die 'Cannot download a file: neither curl nor wget is available!'
	fi
}

download_file() (
	local url="$1"
	local sha256="$2"
	local dest="${3:-.}"

	mkdir -p "$dest" \
		&& cd "$dest" \
		&& rm -f "${url##*/}" \
		&& fetch_url "$url" \
		&& echo "$sha256  ${url##*/}" | sha256sum -c
)

usage() {
	sed -En '/^#---help---/,/^#---help---/p' "$0" | sed -E 's/^# ?//; 1d;$d;'
}

gen_chroot_script() {
	cat <<-EOF
		#!/bin/sh
		set -e

		ENV_FILTER_REGEX='($(echo "$CHROOT_KEEP_VARS" | tr -s ' ' '|'))'
	EOF
	if [ -n "$QEMU_EMULATOR" ]; then
		printf 'export QEMU_EMULATOR="%s"' "$QEMU_EMULATOR"
	fi
	cat <<-'EOF'

		user='root'
		if [ $# -ge 2 ] && [ "$1" = '-u' ]; then
		    user="$2"; shift 2
		fi
		oldpwd="$(pwd)"
		[ "$(id -u)" -eq 0 ] || _sudo='sudo'

		tmpfile="$(mktemp)"
		chmod 644 "$tmpfile"
		export | sed -En "s/^([^=]+ ${ENV_FILTER_REGEX}=)('.*'|\".*\")$/\1\3/p" > "$tmpfile" || true

		cd "$(dirname "$0")"
		$_sudo mv "$tmpfile" env.sh
		$_sudo chroot . /usr/bin/env -i su -l "$user" \
		    sh -c ". /etc/profile; . /env.sh; cd '$oldpwd' 2>/dev/null; \"\$@\"" \
		    -- "${@:-sh}"
	EOF
	# NOTE: ash does not load login profile when run with QEMU user-mode
	# emulation (I have no clue why), that's why /etc/profile is sourced here.
}

gen_destroy_script() {
	cat <<-'EOF'
		#!/bin/sh
		set -e

		remove=no
		case "$1" in
			-r | --remove) remove=yes;;
			'') ;;
			*) echo "Usage: $0 [-r | --remove]"; exit 1;;
		esac

		SCRIPT_DIR=$(cd "$(dirname "$0")" && pwd)
		[ "$(id -u)" -eq 0 ] || _sudo='sudo'

		# Unmounts all filesystem under the specified directory tree.
		cat /proc/mounts | cut -d' ' -f2 | grep "^$SCRIPT_DIR." | sort -r | while read path; do
			echo "Unmounting $path" >&2
			$_sudo umount -fn "$path" || exit 1
		done

		if [ "$remove" = yes ]; then
			rm_opts=''
			# Just to be extra careful...
			rm --help 2>&1 | grep -Fq 'one-file-system' && rm_opts='--one-file-system'

			echo "Removing $SCRIPT_DIR" >&2
			$_sudo rm -Rf $rm_opts "$SCRIPT_DIR"
		else
			echo "If you want to remove $SCRIPT_DIR directory, run: $0 --remove" >&2
		fi
	EOF
}

#------------------------- Debian/Ubuntu ---------------------------#

APT_UPDATED=no

apt_install() {
	if [ "$APT_UPDATED" != yes ]; then
		apt-get update
		APT_UPDATED=yes
	fi
	# -o=Dpkg::Use-Pty=0 - get rid of annoying "Reading database ..." messages.
	apt-get install -y -o=Dpkg::Use-Pty=0 --no-install-recommends "$@"
}

# Installs and enables binfmt-support on Debian/Ubuntu host.
install_binfmt_support() {
	apt_install binfmt-support \
		|| die 'Failed to install binfmt-support using apt-get!'

	update-binfmts --enable \
		|| die 'Failed to enable binfmt!'
}

# Installs QEMU user mode emulation binaries on Debian/Ubuntu host.
install_qemu_user() {
	apt_install qemu-user-static \
		|| die 'Failed to install qemu-user-static using apt-get!'
}


#============================  M a i n  ============================#

while getopts 'a:b:d:i:k:m:p:r:t:hv' OPTION; do
	case "$OPTION" in
		a) ARCH="$OPTARG";;
		b) ALPINE_BRANCH="$OPTARG";;
		d) CHROOT_DIR="$OPTARG";;
		i) BIND_DIR="$OPTARG";;
		k) CHROOT_KEEP_VARS="${CHROOT_KEEP_VARS:-} $OPTARG";;
		m) ALPINE_MIRROR="$OPTARG";;
		p) ALPINE_PACKAGES="${ALPINE_PACKAGES:-} $OPTARG";;
		r) EXTRA_REPOS="${EXTRA_REPOS:-} $OPTARG";;
		t) TEMP_DIR="$OPTARG";;
		h) usage; exit 0;;
		v) echo "alpine-chroot-install $VERSION"; exit 0;;
	esac
done

: ${ALPINE_BRANCH:="latest-stable"}
: ${ALPINE_MIRROR:="http://dl-cdn.alpinelinux.org/alpine"}
: ${ALPINE_PACKAGES:="build-base ca-certificates ssl_client"}
: ${ARCH:=}
: ${BIND_DIR:=}
: ${CHROOT_DIR:="/alpine"}
: ${CHROOT_KEEP_VARS:="ARCH CI QEMU_EMULATOR TRAVIS_.*"}
: ${EXTRA_REPOS:=}
: ${TEMP_DIR:=$(mktemp -d || echo /tmp/alpine)}

# Note: Binding $PWD into chroot as default was a bad idea. It's convenient
# on Travis, but dangerous in general. However, all existing .travis.yml relies
# on this behaviour, so we can't (shouldn't) remove it completely.
[ "$BIND_DIR" ] || case "$(pwd)" in
	/home/*) BIND_DIR="$(pwd)";;
esac


if [ "$(id -u)" -ne 0 ]; then
	die 'This script must be run as root!'
fi

mkdir -p "$CHROOT_DIR"
cd "$CHROOT_DIR"


# Install QEMU user mode emulation if needed (works only on Debian and derivates)

QEMU_EMULATOR=''
if [ -n "$ARCH" ] && [ $(normalize_arch $ARCH) != $(normalize_arch $(uname -m)) ]; then
	qemu_arch="$(normalize_arch $ARCH)"
	QEMU_EMULATOR="/usr/bin/qemu-$qemu_arch-static"

	if [ ! -x "$QEMU_EMULATOR" ]; then
		einfo 'Installing qemu-user-static on host system...'
		install_qemu_user
	fi

	if [ ! -e /proc/sys/fs/binfmt_misc/qemu-$qemu_arch ]; then
		einfo 'Installing and enabling binfmt-support on host system...'
		install_binfmt_support
	fi

	mkdir -p usr/bin
	cp -v "$QEMU_EMULATOR" usr/bin/
fi


einfo 'Downloading static apk-tools'

download_file "$APK_TOOLS_URI" "$APK_TOOLS_SHA256" "$TEMP_DIR"
APK="$TEMP_DIR/apk.static"
chmod +x "$APK"


einfo "Installing Alpine Linux $ALPINE_BRANCH ($ARCH) into chroot"

mkdir -p "$CHROOT_DIR"/etc/apk
cd "$CHROOT_DIR"

printf '%s\n' \
	"$ALPINE_MIRROR/$ALPINE_BRANCH/main" \
	"$ALPINE_MIRROR/$ALPINE_BRANCH/community" \
	$EXTRA_REPOS \
	> etc/apk/repositories

dump_alpine_keys etc/apk/keys/

cp /etc/resolv.conf etc/resolv.conf

"$APK" add \
	--root . --update-cache --initdb --no-progress \
	${ARCH:+--arch $ARCH} \
	alpine-baselayout apk-tools busybox busybox-suid musl-utils

if "$APK" info --root . --no-progress --quiet alpine-release >/dev/null; then
	"$APK" add --root . --no-progress alpine-release
else
	# In Alpine <3.17, this package contains /etc/os-release,
	# /etc/alpine-release and /etc/issue, but we don't wanna install all
	# its dependencies (e.g. openrc).
	"$APK" fetch --root . --no-progress --stdout alpine-base \
		| tar -xz etc
fi

gen_chroot_script > enter-chroot
gen_destroy_script > destroy
chmod +x enter-chroot destroy


einfo 'Binding filesystems into chroot'

mount -v -t proc none proc
mount -v --rbind /sys sys
mount --make-rprivate sys
mount -v --rbind /dev dev
mount --make-rprivate dev

# Some systems (Ubuntu?) symlinks /dev/shm to /run/shm.
if [ -L /dev/shm ] && [ -d /run/shm ]; then
	mkdir -p run/shm
	mount -v --bind /run/shm run/shm
	mount --make-private run/shm
fi

if [ -d "$BIND_DIR" ]; then
	mkdir -p "${CHROOT_DIR}${BIND_DIR}"
	mount -v --bind "$BIND_DIR" "${CHROOT_DIR}${BIND_DIR}"
	mount --make-private "${CHROOT_DIR}${BIND_DIR}"
fi


einfo 'Setting up Alpine'

./enter-chroot <<-EOF
	set -e
	apk update
	apk add $ALPINE_PACKAGES

	if [ -d /etc/sudoers.d ] && [ ! -e /etc/sudoers.d/wheel ]; then
		echo '%wheel ALL=(ALL) NOPASSWD: ALL' > /etc/sudoers.d/wheel
	fi

	if [ -n "${SUDO_USER:-}" ]; then
		adduser -u "${SUDO_UID:-1000}" -G users -s /bin/sh -D "${SUDO_USER:-}" || true
	fi
EOF

cat >&2 <<-EOF
	---
	Alpine installation is complete
	Run $CHROOT_DIR/enter-chroot [-u <user>] [command] to enter the chroot
	and $CHROOT_DIR/destroy [--remove] to destroy it.
EOF
