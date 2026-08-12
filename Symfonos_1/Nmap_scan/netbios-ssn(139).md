139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)

| smb-enum-domains: 
|   Builtin
|     Groups: n/a
|     Users: n/a
|     Creation time: unknown
|     Passwords: min length: 5; min age: n/a days; max age: n/a days; history: n/a passwords
|     Account lockout disabled
|   SYMFONOS
|     Groups: n/a
|     Users: helios
|     Creation time: unknown
|     Passwords: min length: 5; min age: n/a days; max age: n/a days; history: n/a passwords
|_    Account lockout disabled

| smb-enum-shares: 
|   account_used: guest
|   \\192.168.96.18\IPC$: 
|     Type: STYPE_IPC_HIDDEN
|     Comment: IPC Service (Samba 4.5.16-Debian)
|     Users: 1
|     Max Users: <unlimited>
|     Path: C:\tmp
|     Anonymous access: READ/WRITE
|     Current user access: READ/WRITE
|   \\192.168.96.18\anonymous: 
|     Type: STYPE_DISKTREE
|     Comment: 
|     Users: 0
|     Max Users: <unlimited>
|     Path: C:\usr\share\samba\anonymous
|     Anonymous access: READ/WRITE
|     Current user access: READ/WRITE
|   \\192.168.96.18\helios: 
|     Type: STYPE_DISKTREE
|     Comment: Helios personal share
|     Users: 0
|     Max Users: <unlimited>
|     Path: C:\home\helios\share
|     Anonymous access: <none>
|     Current user access: <none>
|   \\192.168.96.18\print$: 
|     Type: STYPE_DISKTREE
|     Comment: Printer Drivers
|     Users: 0
|     Max Users: <unlimited>
|     Path: C:\var\lib\samba\printers
|     Anonymous access: <none>
|_    Current user access: <none>

| smb-enum-users: 
|   SYMFONOS\helios (RID: 1000)
|     Full name:   
|     Description: 
|_    Flags:       Normal user account
