

# By terminal

#### 1. check your volumes states, then remember the disk you wanna to give write permission. here we get "BOOT CAMP" 
```
diskutil list
```

#### 2. open or create /etc/fstab file.
```
sudo nano /etc/fstab
```

#### 3. paste the text below in terminal, replace the "BOOT CAMP" with your disk name, if your disk name has space characters, use /040 replace them. then ctrl+X, Y
```
LABEL=BOOT/040CAMP none ntfs rw,auto,nobrowse
```

#### 4. restart your mac.
