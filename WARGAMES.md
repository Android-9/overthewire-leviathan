## Leviathan
#### Level 0
Login to the first level with the credentials:

> Host: **leviathan.labs.overthewire.org** <br>
> Port: 2223 <br>
> Username: **leviathan0** <br>
> Password: **leviathan0** <br>

`ssh leviathan0@leviathan.labs.overthewire.org -p 2223`

---

#### Level 1
If you try and list all normal files in the root directory you will find there are none. However, listing all files including hidden ones will net you the usual `.bash_logout`, `.bashrc` and `.profile` files, but also another one called `.backup`.

You will find a `bookmarks.html` file within the `.backup` directory.

Reading the entire file with `cat` will yield a massive heap of results.

One might find it natural to use `grep` to search for the word 'password' in the HTML document.

`grep "password" bookmarks.html`

```bash
leviathan0@gibson:~/.backup$ grep "password" bookmarks.html
<DT><A HREF="http://leviathan.labs.overthewire.org/passwordus.html | This will be fixed later, the password for leviathan1 is 3QJ3TgzHDq" ADD_DATE="1155384634" LAST_CHARSET="ISO-8859-1" ID="rdf:#$2wIU71">password to leviathan1</A>
```

Password: 3QJ3TgzHDq

---

#### Level 2


---

#### Level 3


---

#### Level 4


---

#### Level 5


---

#### Level 6


---

#### Level 7
