# SQL Server 2014 TLS/SChannel Startup Issue - Incident Report

## Triệu chứng ban đầu

Dịch vụ **MSSQLSERVER** không khởi động được.

### SQL Error Log

``` text
Error: 26011
The server was unable to initialize encryption

Error: 17182
TDSSNIClient initialization failed
Unable to initialize SSL support

Error: 17826
Could not start the network library
```

### Windows Event Viewer

``` text
Event ID: 36871
Source: Schannel

A fatal error occurred while creating a TLS client credential.
The internal error state is 10013.
```

------------------------------------------------------------------------

## Nguyên nhân

Server đã tắt hoàn toàn:

``` text
TLS 1.0
TLS 1.1
```

Chỉ còn:

``` text
TLS 1.2
```

SQL Server 2014 SP1 (12.0.4100.1) không khởi tạo được SSL/TLS nên không
thể mở Network Library.

------------------------------------------------------------------------

## Bước 1 - Kiểm tra TLS

``` cmd
reg query "HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols" /s
```

------------------------------------------------------------------------

## Bước 2 - Bật lại TLS 1.0

``` cmd
reg add "HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.0\Client" /v Enabled /t REG_DWORD /d 1 /f

reg add "HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.0\Client" /v DisabledByDefault /t REG_DWORD /d 0 /f

reg add "HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.0\Server" /v Enabled /t REG_DWORD /d 1 /f

reg add "HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.0\Server" /v DisabledByDefault /t REG_DWORD /d 0 /f
```

### (Tùy chọn) Bật TLS 1.1

``` cmd
reg add "HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.1\Client" /v Enabled /t REG_DWORD /d 1 /f

reg add "HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.1\Client" /v DisabledByDefault /t REG_DWORD /d 0 /f

reg add "HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.1\Server" /v Enabled /t REG_DWORD /d 1 /f

reg add "HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.1\Server" /v DisabledByDefault /t REG_DWORD /d 0 /f
```

------------------------------------------------------------------------

## Bước 3 - Khởi động lại máy chủ

``` cmd
shutdown /r /t 0
```

Trong quá trình khởi động lại có thể xuất hiện:

``` text
Working on updates 100% complete
Don't turn off your computer
```

------------------------------------------------------------------------

## Bước 4 - Kiểm tra SQL Server

``` cmd
sc query MSSQLSERVER
```

Kết quả mong muốn:

``` text
STATE : 4 RUNNING
```

------------------------------------------------------------------------

## Kết quả

Sau khi:

-   Bật lại TLS 1.0
-   Reboot server
-   SQL Server khởi động thành công

=\> Xác định nguyên nhân là cấu hình TLS/SChannel quá chặt khiến SQL
Server 2014 không khởi tạo được SSL/TLS.

------------------------------------------------------------------------

## Ghi chú

Nên:

1.  Backup toàn bộ database.
2.  Kiểm tra lại TLS/Cipher Suite.
3.  Theo dõi Event ID 36871.
4.  Lập kế hoạch nâng cấp SQL Server hoặc cập nhật đầy đủ hỗ trợ TLS
    1.2.

------------------------------------------------------------------------

## Sự cố phát sinh sau đó

``` cmd
sqlcmd -S localhost -E
```

Trả về:

``` text
Login failed for user 'WIN-IU72E44JAUE\Administrator'.
Reason: The account is disabled.
```

Ý nghĩa:

-   SQL Server đã chạy bình thường.
-   TLS đã hoạt động.
-   Tài khoản Windows Administrator trong SQL bị Disable.
-   Đây là lỗi phân quyền đăng nhập, không còn là lỗi TLS.
