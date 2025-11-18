
Если push падает из-за большого blob’а, увеличь лимит:

```bash
git config http.postBuffer 524288000
```

или подумай, нужен ли вообще этот файл в репо.  
Для медиа/бинарников используй [Git LFS](https://git-lfs.github.com/).