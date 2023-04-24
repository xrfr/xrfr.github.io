---
title: Connect google docs notes to markdown files
feed: hide
---

- [ ] @write this up into a proper blog post

- rclone config was inside the encfs store, but it looks like rclone offers it's own encryption
- we can store `.rclone.conf` directly inside dotfiles repo 

Now we can... this shows us `.docx` files

```sh
rclone ls gdrive:folder/
```

Pipe into pandoc and we we see markdown!
```
rclone cat sachin.rudraraju:znotebook/Roadmap.docx | pandoc --from docx --to markdown | less
```

We can generate docx from markdown and reupload back to google docs

```
pandoc test.md --reference-doc ~/Downloads/custom-reference.docx -o test.docx
```

- [ ] for some reason, can't edit the reference-doc in Google Docs, might need to edit it in Word directly according to following 
  - https://stackoverflow.com/questions/70513062/how-do-i-add-custom-formatting-to-docx-files-generated-in-pandoc
- keep default formatting for now

- automate this for each file
- [x] for each file in sync folder
  - [x] created `sync-settings.yaml` with user friendly format
  - [x] stash local changes
  - [x] copy file from google drive and convert to markdown
  - [x] pop local changes
  - [x] create docx file and copy to google drive