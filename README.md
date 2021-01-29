# BUG Rclone union

## Details
Renaming a file from a read-only upstream on a server trigger error but still list the file has renamed even if not accessible.

```
./rclone -V
rclone v1.53.4
- os/arch: linux/amd64
- go version: go1.15.6
```

## To reproduce

### start a local server with a predefined user and pass
```
./rclone --config=./rclone.conf serve sftp --user test --pass test union:
```

### local fs look like that:
```
data/
├── bottom
│   └── file_bottom_1
└── top
    └── file_top_1
```


### Rename a file from bottom (read-only) 
Using nautilus (gvfs) rename file_bottom_1 to file_bottom_2

Rclone reject the request and we received an error
```
2021/01/29 14:45:29 ERROR : file_bottom_1: Couldn't move: permission denied
2021/01/29 14:45:29 ERROR : file_bottom_2: File.Rename error: permission denied
2021/01/29 14:45:29 ERROR : file_bottom_1: Dir.Rename error: permission denied
```

### Result
Local fs is not changed
```
data/
├── bottom
│   └── file_bottom_1
└── top
    └── file_top_1
```

But rclone expose on sftp as the file is renamed.
```
$ ls -lah .
ls: impossible d'accéder à 'file_bottom_2': Aucun fichier ou dossier de ce type
total 0
drwxr-xr-x 1 XXX XXX 0 janv. 29 14:43 .
dr-x------ 3 XXX XXX 0 janv. 27 11:00 ..
?????????? ? ?       ?                       ?              ? file_bottom_2
-rw-r--r-- 1 XXX XXX 0 janv. 29 14:42 file_top_1
```

We can read (cat command) the file_bottom_1 even if it is not listed.
We can't read file_bottom_2.

This seems related to a cache issue of the rename request.

## Expectations

Two possibility:
- Rclone should not list the file file_bottom_2 and display file_bottom_1 after the rejection (keep previous state).
- Rclone should create a duplicate file in writable upstream and list both.

I feel like the first is the best solution.

Note: using mv compared to renaming from nautilus (gvfs) create a file_bottom_2 (with data of file_bottom_1) in top upstream but it may be related on how mv works and result on the same bug (file 2 listed but not readable and file 1 not listed but readable).

