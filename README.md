# Firmware-Entry-Points-In-Memory
A research paper about auto detecting the base address and entry points of firmware in memory.

To Do:             
1) Disassemble compiled Binary - use x86 binary in ```openwrt``` directory      
    - How: Binary Ninja (preferred), Radare2, objdump
    
2) Find out how to determine where placed in memory

3) Create script to automate task

Current References:              
[1] Attacks on Fitness Trackers Revisited: A Case-Study of Unfit Firmware Security
https://arxiv.org/pdf/1604.03313.pdf

[2] Firmalice - Automatic Detection of Authentication Bypass Vulnerabilities in Binary Firmware
https://www.lastline.com/papers/2015_ndss15_firmalice-2.pdf

[3] Weighing in on Issues with “Cloud Scale”
https://mncoppola.files.wordpress.com/2013/06/weighing-in-on-issues-with-cloud-scale.pdf

[4] Static Disassembly of Obfuscated Binaries
https://www.usenix.org/legacy/publications/library/proceedings/sec04/tech/full_papers/kruegel/kruegel_html/disassemble.html

[5] Avatar: A Framework to Support Dynamic Security Analysis of Embedded Systems’ Firmwares
http://www.eurecom.fr/en/publication/4158/download/rs-publi-4158.pdf

[6] A Large-Scale Analysis of the Security of Embedded Firmwares
https://www.usenix.org/system/files/conference/usenixsecurity14/sec14-paper-costin.pdf

[7] http://binwalk.org/

[8] http://www.devttys0.com/2014/02/reversing-the-wrt120n-firmware-obfuscation/

[9] https://sviehb.wordpress.com/2011/09/06/reverse-engineering-an-obfuscated-firmware-image-e01-unpacking/

[10] When Firmware Modifications Attack: A Case Study of Embedded Exploitation
http://blog.narotama.ac.id/wp-content/uploads/2014/12/When-Firmware-Modifications-Attack-A-Case-Study-of-Embedded-Exploitation.pdf

[11] Analysis of programmable logic controller firmware for threat assessment and forensic investigation
https://books.google.com/books?hl=en&lr=&id=s7kVBAAAQBAJ&oi=fnd&pg=PA9&dq=firmware+header&ots=kUzInk-KsR&sig=mTGgWzNXX--UWTt4dULFxRFng8U#v=onepage&q&f=false

[12] Embedded Devices Security Firmware Reverse Engineering
Jonas Zaddach | Andrei Costin
https://media.blackhat.com/us-13/US-13-Zaddach-Workshop-on-Embedded-Devices-Security-and-Firmware-Reverse-Engineering-Slides.pdf

[13] Automated Dynamic Firmware Analysis at Scale: A Case Study on Embedded Web Interfaces
http://s3.eurecom.fr/docs/dynamicanalysis_costin.pdf

[14] FIE on Firmware: Finding Vulnerabilities in Embedded Systems using Symbolic Execution
https://www.usenix.org/system/files/conference/usenixsecurity13/sec13-paper_davidson.pdf

[15] KLEE: Unassisted and Automatic Generation of High-Coverage Tests for Complex Systems Programs
http://hci.stanford.edu/cstr/reports/2008-03.pdf