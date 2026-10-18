# How to move WSL to another disk

- Move Ubuntu to disk D.

```powershell
cd D:\
mkdir WSL
cd WSL
wsl --export Ubuntu ubuntu.tar
wsl --unregister Ubuntu
mkdir Ubuntu
wsl --import Ubuntu Ubuntu ubuntu.tar
```

## References

- iany / [Move WSL to Another Drive](https://blog.iany.me/2020/06/move-wsl-to-another-drive/)
