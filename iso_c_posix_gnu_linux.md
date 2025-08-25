### **標準 C (ISO C 標準庫)** 

### **POSIX / GNU 擴充 (Unix/Linux 常見)**

### **Linux kernel 專用 API**



# C 語言字串處理完整總表 (含 GNU / POSIX / Linux kernel)  
## 1️⃣ 標準 C (ISO C 標準)  


| 函式                                             | 功能                  | header       |
| ---------------------------------------------- | ------------------- | ------------ |
| `strlen`                                       | 回傳字串長度              | `<string.h>` |
| `strcpy`, `strncpy`                            | 複製字串                | `<string.h>` |
| `strcat`, `strncat`                            | 串接字串                | `<string.h>` |
| `strcmp`, `strncmp`                            | 比較字串                | `<string.h>` |
| `strchr`, `strrchr`                            | 尋找字元（正向/反向）         | `<string.h>` |
| `strstr`                                       | 尋找子字串               | `<string.h>` |
| `strpbrk`                                      | 尋找第一個匹配字元           | `<string.h>` |
| `strspn`, `strcspn`                            | 計算前綴長度              | `<string.h>` |
| `strtok`                                       | 切割字串（非安全，多次呼叫共享靜態區） | `<string.h>` |
| `memcpy`, `memmove`                            | 記憶體區塊拷貝             | `<string.h>` |
| `memcmp`                                       | 記憶體比較               | `<string.h>` |
| `memset`                                       | 記憶體初始化              | `<string.h>` |
| `memchr`                                       | 記憶體搜尋字元             | `<string.h>` |
| `strerror`                                     | 錯誤碼轉字串              | `<string.h>` |
| `atoi`, `atol`, `atoll`                        | 字串轉整數               | `<stdlib.h>` |
| `atof`                                         | 字串轉浮點數              | `<stdlib.h>` |
| `strtol`, `strtoul`, `strtoll`, `strtoull`     | 進位制字串轉數字            | `<stdlib.h>` |
| `strtod`, `strtof`, `strtold`                  | 字串轉浮點數（安全）          | `<stdlib.h>` |
| `sprintf`, `snprintf`                          | 格式化輸出到字串            | `<stdio.h>`  |
| `sscanf`                                       | 從字串讀取資料             | `<stdio.h>`  |
| `fgets`, `puts`, `fputs`                       | 輸入輸出字串              | `<stdio.h>`  |
| `toupper`, `tolower`                           | 轉換字元大小寫             | `<ctype.h>`  |
| `isalpha`, `isdigit`, `isalnum`, `isspace` ... | 字元判斷                | `<ctype.h>`  |


## 2️⃣ POSIX / GNU 擴充

**這些不是 ISO C 標準，但在 Unix/Linux/Glibc 幾乎都會用**

| 函式                      | 功能                                            | header        |
| ----------------------- | --------------------------------------------- | ------------- |
| `strcasecmp`            | 不分大小寫比較字串                                     | `<strings.h>` |
| `strncasecmp`           | 不分大小寫比較（前 N 個字元）                              | `<strings.h>` |
| `strdup`                | 建立字串副本 (malloc)                               | `<string.h>`  |
| `strndup`               | 建立字串副本（指定長度）                                  | `<string.h>`  |
| `strsignal`             | 錯誤訊號轉字串 (e.g. SIGSEGV → "Segmentation fault") | `<string.h>`  |
| `strtok_r`              | 可重入版字串切割 (thread-safe)                        | `<string.h>`  |
| `strchrnul`             | 找字元（找不到回傳 `\0` 的位置）                           | `<string.h>`  |
| `stpcpy`, `stpncpy`     | 複製字串並回傳結尾指標                                   | `<string.h>`  |
| `mempcpy`               | 拷貝記憶體並回傳結尾指標                                  | `<string.h>`  |
| `strlcpy`, `strlcat`    | 安全複製/串接字串 (BSD/GNU)                           | `<string.h>`  |
| `asprintf`, `vasprintf` | 動態配置並格式化字串                                    | `<stdio.h>`   |
| `getline`, `getdelim`   | 動態讀取字串 (會 malloc)                             | `<stdio.h>`   |
| `strfry`                | 隨機打亂字串 (glibc)                                | `<string.h>`  |
| `memmem`                | 在記憶體區找子序列 (glibc)                             | `<string.h>`  |


## 3️⃣ Linux Kernel 專用 (不可直接用 libc)

**Linux kernel 不能用 <string.h> 的 libc 版本，必須用 內核提供的 API，這些都在 #include <linux/string.h> 或 #include <linux/kernel.h>**

| 函式                                                                        | 功能                                      | header             |
| ------------------------------------------------------------------------- | --------------------------------------- | ------------------ |
| `strlen`                                                                  | 計算字串長度                                  | `<linux/string.h>` |
| `strnlen`                                                                 | 最多算 N 個字元                               | `<linux/string.h>` |
| `strcpy`, `strncpy`                                                       | 複製字串                                    | `<linux/string.h>` |
| `strlcpy`                                                                 | 安全複製字串                                  | `<linux/string.h>` |
| `strcat`, `strncat`                                                       | 串接字串                                    | `<linux/string.h>` |
| `strcmp`, `strncmp`                                                       | 比較字串                                    | `<linux/string.h>` |
| `strcasecmp`, `strncasecmp`                                               | 不分大小寫比較                                 | `<linux/string.h>` |
| `strchr`, `strrchr`                                                       | 尋找字元                                    | `<linux/string.h>` |
| `strstr`                                                                  | 尋找子字串                                   | `<linux/string.h>` |
| `memset`, `memcpy`, `memmove`, `memcmp`                                   | 記憶體操作                                   | `<linux/string.h>` |
| `memchr`                                                                  | 記憶體尋找字元                                 | `<linux/string.h>` |
| `memscan`                                                                 | 尋找第一個不符合字元                              | `<linux/string.h>` |
| `strscpy`                                                                 | **Kernel 專用安全字串複製 API** (比 strncpy 更安全) | `<linux/string.h>` |
| `kstrtoint`, `kstrtouint`, `kstrtoul`, `kstrtoull`, `kstrtol`, `kstrtoll` | kernel 字串轉整數 API                        | `<linux/kernel.h>` |
| `kstrto*` 系列                                                              | 安全字串 → 數字轉換 (支援進位制)                     | `<linux/kernel.h>` |
| `kasprintf`, `kvasprintf`                                                 | 動態分配並格式化字串 (kmalloc)                    | `<linux/slab.h>`   |



✅ 總結建議

如果你在 用一般 C 程式 (User space) → 以 <string.h> + <ctype.h> + <stdlib.h> + <stdio.h> 為主，再補上 GNU/POSIX 的 strings.h、asprintf、getline。

如果你在 寫 Linux kernel module / driver → 禁止用 libc，必須用 <linux/string.h> 與 <linux/kernel.h> 提供的函式，特別是 strscpy、kstrto*、kasprintf 這些安全版本。