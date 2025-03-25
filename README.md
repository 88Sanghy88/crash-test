#### **Desctiption**

A Null Pointer Dereference has occurred when running program bsdtar in function header_pax_extension at  rchive_read_support_format_tar.c:1844:8.

#### Version

Libarchive-3.7.6,Libarchive-3.7.5 and before .

#### **Steps to reproduce**

build

```bash
wget https://github.com/libarchive/libarchive/releases/download/v3.7.6/libarchive-3.7.6.tar.gz
CC=clang-14 CFLAGS="-g -fsanitize=address -fno-omit-frame-pointer" CXXFLAGS="-g -fsanitize=address -fno-omit-frame-pointer" ./configure --prefix=`pwd`/re-build
make
make install
```

reproduce

```bash
xx@xx-virtual-machine:~/fuzz_libarchive/libarchive-3.7.6$ re-build/bin/bsdtar -xf ./poc
AddressSanitizer:DEADLYSIGNAL
=================================================================
==853649==ERROR: AddressSanitizer: SEGV on unknown address 0x000000000000 (pc 0x5c15d16f52e3 bp 0x7fffb2638530 sp 0x7fffb26380c0 T0)
==853649==The signal is caused by a READ memory access.
==853649==Hint: address points to the zero page.
    #0 0x5c15d16f52e3 in header_pax_extension /home/xx/fuzz_libarchive/libarchive-3.7.6/libarchive/archive_read_support_format_tar.c:1844:8
    #1 0x5c15d16f2f2e in tar_read_header /home/xx/fuzz_libarchive/libarchive-3.7.6/libarchive/archive_read_support_format_tar.c:869:11
    #2 0x5c15d16efa38 in archive_read_format_tar_read_header /home/xx/fuzz_libarchive/libarchive-3.7.6/libarchive/archive_read_support_format_tar.c:550:6
    #3 0x5c15d162953b in _archive_read_next_header2 /home/xx/fuzz_libarchive/libarchive-3.7.6/libarchive/archive_read.c:646:7
    #4 0x5c15d16291f4 in _archive_read_next_header /home/xx/fuzz_libarchive/libarchive-3.7.6/libarchive/archive_read.c:684:8
    #5 0x5c15d173c52f in archive_read_next_header /home/xx/fuzz_libarchive/libarchive-3.7.6/libarchive/archive_virtual.c:148:10
    #6 0x5c15d15f8219 in read_archive /home/xx/fuzz_libarchive/libarchive-3.7.6/tar/read.c:259:7
    #7 0x5c15d15f9974 in tar_mode_x /home/xx/fuzz_libarchive/libarchive-3.7.6/tar/read.c:111:2
    #8 0x5c15d15f3bbf in main /home/xx/fuzz_libarchive/libarchive-3.7.6/tar/bsdtar.c:1008:3
    #9 0x7b21b5029d8f in __libc_start_call_main csu/../sysdeps/nptl/libc_start_call_main.h:58:16
    #10 0x7b21b5029e3f in __libc_start_main csu/../csu/libc-start.c:392:3
    #11 0x5c15d153a934 in _start (/home/xx/fuzz_libarchive/libarchive-3.7.6/re-build/bin/bsdtar+0x73934) (BuildId: 553ff155759af8fb2e57e5ce688d0b556085610b)

AddressSanitizer can not provide additional info.
SUMMARY: AddressSanitizer: SEGV /home/xx/fuzz_libarchive/libarchive-3.7.6/libarchive/archive_read_support_format_tar.c:1844:8 in header_pax_extension
==853649==ABORTING

```

#### POC

https://github.com/88Sanghy88/crash-test/blob/main/poc

#### code in function header_pax_extension at rchive_read_support_format_tar.c:1844:8

```c
	while (ext_size > 0) {
		/* Read enough bytes to parse the size/name of the next attribute */
		to_read = max_size_name;
		if (to_read > ext_size) {
			to_read = ext_size;
		}
		p = __archive_read_ahead(a, to_read, &did_read);    // line 1820,here may return NULL.
		if (did_read < 0) {
			return ((int)did_read);
		}
		if (did_read == 0) { /* EOF */
			archive_set_error(&a->archive, EINVAL,
					  "Truncated tar archive"
					  " detected while reading pax attribute name");
			return (ARCHIVE_FATAL);
		}
		if (did_read > ext_size) {
			did_read = ext_size;
		}

		/* Parse size of attribute */
		line_length = 0;
		attr_start = p;
		while (1) {
			if (p >= attr_start + did_read) {
				archive_set_error(&a->archive, ARCHIVE_ERRNO_MISC,
						  "Ignoring malformed pax attributes: overlarge attribute size field");
				*unconsumed += ext_size + ext_padding;
				return (ARCHIVE_WARN);
			}
			if (*p == ' ') {       //line 1844 , trigger
				p++;
				break;
			}
```

#### **Environment**

```bash
Linux xx-virtual-machine 6.8.0-40-generic #40~22.04.3-Ubuntu SMP PREEMPT_DYNAMIC Tue Jul 30 17:30:19 UTC 2 x86_64 x86_64 x86_64 GNU/Linux
```



